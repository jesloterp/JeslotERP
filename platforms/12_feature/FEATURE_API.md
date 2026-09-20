# JeslotERP Feature Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — flag/override HTTP persists on Postgres; empty list is `[]`. Evaluate stays CatalogStore. Not Production.  
**Package:** `platforms.p12_feature`  
**PostgreSQL schema:** `feature`  
**Public base:** `/api/v1/features`  
**Internal base:** `/internal/v1/features`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`FEATURE_GUIDE.md`](FEATURE_GUIDE.md) · [`FEATURE_SCHEMA.md`](FEATURE_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Evaluate/bootstrap/explain, targeting, segments, rollouts, kill switch, overrides, schedules, experiments, promote, packs. |
| **1.0 SoR-Live** | **2026-09-12** | GET `/flags`, `/kills`, `/overrides` are Postgres-first; PUT/DELETE overrides dual-write. Evaluate/bootstrap do not read the ledger. |

---

## 1. Design principles (advanced)

1. **Evaluate-first** — apps use `/evaluate` and `/bootstrap`; admin APIs are secondary.  
2. **Server authoritative** — client bootstrap is a hint; sensitive paths re-evaluate server-side.  
3. **Explainable** — every evaluation can return a reason code.  
4. **Environment explicit** — `env` query/header required (or default from deployment).  
5. **Kill switch wins** — fastest incident path.  
6. **Sticky bucketing** — stable % experience.  
7. **ETag bootstrap** — `If-None-Match` → 304.  
8. **Idempotent kill/override/schedule**.  
9. **Production guardrails** — optional approval for rule changes.  
10. **No silent type coercion errors** — bad context attrs → clause false or ERROR_DEFAULT per policy.  
11. **Exposure async** — does not block evaluate latency SLO.  
12. **License facts optional** — pass in context; p12 does not replace p26.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Company-Id: <uuid>
X-Feature-Env: production
If-None-Match: "<etag>"
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {
    "request_id": "…",
    "etag": "W/\"feat-boot-…\"",
    "env": "production"
  }
}
```

---

## 4. Errors

```text
FEAT_FLAG_NOT_FOUND / ENV_NOT_FOUND / SEGMENT_NOT_FOUND
FEAT_VARIATION_NOT_FOUND / INVALID_FLAG_TYPE
FEAT_RULE_INVALID / ROLLOUT_WEIGHTS_INVALID
FEAT_PREREQ_CYCLE / PREREQ_FAILED
FEAT_KILL_ACTIVE / OVERRIDE_CONFLICT / OVERRIDE_EXPIRED
FEAT_SCHEDULE_INVALID / SCHEDULE_NOT_PENDING
FEAT_EXPERIMENT_INVALID / EXPERIMENT_NOT_RUNNING
FEAT_APPROVAL_REQUIRED / PUBLISH_CONFLICT
FEAT_PACKAGE_CHECKSUM_MISMATCH
FEAT_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
FEAT_SDK_UNAUTHORIZED
FEAT_BOOTSTRAP_UNAUTHORIZED
```

HTTP: `404` · `409` · `422` · `403` · `412`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `feature.catalog.read` | Read flags |
| `feature.catalog.manage` | Mutate flags/variations |
| `feature.targeting.manage` | Rules/segments/rollouts |
| `feature.override.manage` | Overrides |
| `feature.kill` | Kill switch |
| `feature.publish` | Promote/schedule apply |
| `feature.evaluate` | Evaluate/bootstrap |
| `feature.experiment.manage` | Experiments |
| `feature.pack.install` | Packs |
| `feature.audit.read` | Audit |
| `feature.*` | All |

---

## 6. Evaluate APIs (primary runtime)

### 6.1 Evaluate one / many

```http
POST /api/v1/features/evaluate
```

```json
{
  "env": "production",
  "flag_keys": [
    "sales.order.bulk_import.v2",
    "ui.shell.dense_mode"
  ],
  "context": {
    "tenant_id": "…",
    "company_id": "…",
    "user_id": "…",
    "roles": ["OPS_MANAGER"],
    "plan_code": "ENTERPRISE",
    "license_modules": ["transport.advanced"],
    "attributes": { "region": "IN-WEST", "beta_tester": true },
    "bucket_key": "user:…"
  },
  "explain": true,
  "record_exposure": true
}
```

**Response:**

```json
{
  "env": "production",
  "flags": {
    "sales.order.bulk_import.v2": {
      "value": true,
      "variation_key": "on",
      "reason": "RULE_MATCH",
      "rule_id": "…",
      "explain": [
        { "step": "kill_switch", "active": false },
        { "step": "prereq", "ok": true },
        { "step": "override", "matched": false },
        { "step": "rule", "position": 0, "matched": true }
      ]
    },
    "ui.shell.dense_mode": {
      "value": "dense",
      "variation_key": "dense",
      "reason": "PERCENT_ROLLOUT",
      "bucket": 7341
    }
  }
}
```

Omit `flag_keys` to evaluate all `client_side_available` flags (bounded) — prefer explicit keys.

### 6.2 Evaluate single (GET sugar)

```http
GET /api/v1/features/flags/{flag_key}/evaluate?env=production&explain=true
```

Context from JWT + query attrs (`company_id`, custom `attr.*`).

### 6.3 Bootstrap (SPA shell)

```http
GET /api/v1/features/bootstrap?env=production
If-None-Match: W/"feat-boot-…"
```

Returns client-safe flag map + etag. `304` when unchanged.  
Only flags with `client_side_available=true`.

### 6.4 Internal evaluate

```http
POST /internal/v1/features/evaluate
POST /internal/v1/features/evaluate-all-for-context
```

For workers, rules context provider, API gateways. Requires internal token + explicit tenant.

---

## 7. Flag catalog

```http
GET    /api/v1/features/flags
POST   /api/v1/features/flags
GET    /api/v1/features/flags/{flag_key}
PATCH  /api/v1/features/flags/{flag_key}
POST   /api/v1/features/flags/{flag_key}/archive
```

**Create:**

```json
{
  "flag_key": "sales.order.bulk_import.v2",
  "name": "Bilty bulk import v2",
  "flag_type": "BOOLEAN",
  "is_temporary": true,
  "client_side_available": false,
  "variations": [
    { "variation_key": "off", "value": false, "is_off_variation": true },
    { "variation_key": "on", "value": true }
  ]
}
```

### Variations

```http
GET  /api/v1/features/flags/{flag_key}/variations
POST /api/v1/features/flags/{flag_key}/variations
PATCH /api/v1/features/flags/{flag_key}/variations/{variation_key}
```

Do not change semantic meaning of an existing variation value in production — add a new variation.

---

## 8. Environments & targeting

### 8.1 Env list / config

```http
GET /api/v1/features/environments
GET /api/v1/features/flags/{flag_key}/envs/{env_key}
PUT /api/v1/features/flags/{flag_key}/envs/{env_key}
```

**Put config (If-Match version):**

```json
{
  "is_enabled": true,
  "rules": [
    {
      "position": 0,
      "description": "Beta tenants",
      "segment_key": "beta_tenants",
      "serve": { "kind": "VARIATION", "variation_key": "on" }
    },
    {
      "position": 1,
      "description": "West region enterprise",
      "clauses": [
        { "attribute": "plan_code", "op": "EQ", "value": "ENTERPRISE" },
        { "attribute": "attributes.region", "op": "EQ", "value": "IN-WEST" }
      ],
      "serve": { "kind": "VARIATION", "variation_key": "on" }
    }
  ],
  "fallthrough": {
    "kind": "ROLLOUT",
    "bucket_by": "user_id",
    "weights": [
      { "variation_key": "off", "weight_bps": 9000 },
      { "variation_key": "on", "weight_bps": 1000 }
    ]
  },
  "prerequisites": [
    { "flag_key": "transport.module.enabled", "variation_key": "on" }
  ]
}
```

Production may return `FEAT_APPROVAL_REQUIRED` and create a changeset instead of direct apply.

### 8.2 Patch helpers

```http
POST /api/v1/features/flags/{flag_key}/envs/{env_key}/rules
PATCH /api/v1/features/flags/{flag_key}/envs/{env_key}/rules/{rule_id}
DELETE /api/v1/features/flags/{flag_key}/envs/{env_key}/rules/{rule_id}
PUT /api/v1/features/flags/{flag_key}/envs/{env_key}/fallthrough
PUT /api/v1/features/flags/{flag_key}/envs/{env_key}/prerequisites
```

---

## 9. Segments

```http
GET    /api/v1/features/segments
POST   /api/v1/features/segments
GET    /api/v1/features/segments/{segment_key}
PUT    /api/v1/features/segments/{segment_key}
PUT    /api/v1/features/segments/{segment_key}/members
POST   /api/v1/features/segments/{segment_key}/members:batch
```

**Members batch:**

```json
{
  "include": [{ "member_type": "TENANT", "member_id": "…" }],
  "exclude": []
}
```

---

## 10. Kill switch

```http
POST /api/v1/features/flags/{flag_key}/kill
POST /api/v1/features/flags/{flag_key}/kill/clear
GET  /api/v1/features/kills
```

**Kill body:**

```json
{
  "env": "production",
  "reason": "Error spike on bulk import"
}
```

Requires `feature.kill`. Emits `feature.kill.engaged`. Evaluate reason becomes `KILL_SWITCH`.

**Break-glass (rare):**

```http
POST /api/v1/features/flags/{flag_key}/break-glass
```

Allows named subjects during kill — heavily audited.

---

## 11. Overrides

```http
GET    /api/v1/features/overrides?flag_key=…&tenant_id=…
PUT    /api/v1/features/overrides/tenant
PUT    /api/v1/features/overrides/company
DELETE /api/v1/features/overrides/{override_id}
```

```json
{
  "flag_key": "sales.order.bulk_import.v2",
  "env": "production",
  "tenant_id": "…",
  "mode": "FORCE_ON",
  "reason_code": "SUPPORT_ENABLE",
  "expires_at": "2026-09-16T00:00:00Z"
}
```

---

## 12. Schedules & promote

```http
GET  /api/v1/features/schedules
POST /api/v1/features/schedules
POST /api/v1/features/schedules/{id}/cancel
GET  /api/v1/features/schedules/{id}/runs

POST /api/v1/features/promote
GET  /api/v1/features/promote/{id}
POST /api/v1/features/promote/{id}/apply
```

**Schedule:**

```json
{
  "flag_key": "ui.shell.dense_mode",
  "env": "production",
  "execute_at": "2026-09-10T02:00:00Z",
  "action": {
    "type": "SET_FALLTHROUGH_PERCENT",
    "weights": [
      { "variation_key": "off", "weight_bps": 5000 },
      { "variation_key": "dense", "weight_bps": 5000 }
    ]
  }
}
```

**Promote:** `{ "flag_key": "…", "from_env": "staging", "to_env": "production" }` creates diff + approval.

Internal tick:

```http
POST /internal/v1/features/schedules/tick
```

---

## 13. Experiments

```http
GET  /api/v1/features/experiments
POST /api/v1/features/experiments
POST /api/v1/features/experiments/{key}/start
POST /api/v1/features/experiments/{key}/stop
GET  /api/v1/features/experiments/{key}/exposures/summary
```

Exposures are written on evaluate when `record_exposure=true` and experiment RUNNING.

---

## 14. Packages & governance

```http
GET  /api/v1/features/packages
POST /api/v1/features/packages/{package_key}/install
GET  /api/v1/features/changesets
POST /api/v1/features/changesets/{id}/approvals
```

---

## 15. Audit & stats

```http
GET /api/v1/features/flags/{flag_key}/audit
GET /api/v1/features/stats?flag_key=…&from=…&to=…
GET /api/v1/features/eval-samples?flag_key=…
```

Requires `feature.audit.read` for detailed samples.

---

## 16. Webhooks & SDK keys (admin)

```http
GET  /api/v1/features/webhooks
POST /api/v1/features/webhooks
GET  /api/v1/features/sdk-keys
POST /api/v1/features/sdk-keys
```

SDK secrets stored via configuration `secret_ref`; create returns secret once.

---

## 17. Caching & concurrency

| Resource | Strategy |
|---|---|
| Env flag config | Version + checksum; cache by flag+env |
| Bootstrap | ETag by context scope hash |
| Kill switch | Ultra-hot cache with immediate invalidate |
| Evaluate | Sticky optional durable only when configured |
| Config updates | If-Match on `version` |

---

## 18. Example client flows

### 18.1 Progressive rollout

1. Create boolean flag with off/on  
2. Staging: target beta segment → on  
3. Promote to production with approval  
4. Fallthrough 10% on; schedule bump to 50% overnight  
5. Incident → kill; fix; clear kill; resume %  

### 18.2 SPA boot

1. Login  
2. `GET /bootstrap` with ETag  
3. Render nav gated by flags  
4. Server API still evaluates sensitive flags on request  

### 18.3 Support enables tenant

1. `PUT /overrides/tenant` FORCE_ON with expiry  
2. Customer verifies  
3. Override expires or deleted  

### 18.4 Rules/process context

1. p11 provider `feature.flags` calls internal evaluate  
2. Decision table uses `flags.sales.order.bulk_import.v2` fact  

---

## 19. Event hooks

| Event | Consumer |
|---|---|
| `feature.kill.engaged` | Status page / on-call |
| `feature.env.rules.changed` | Cache bust / CDN |
| `feature.override.changed` | Support audit |
| `feature.exposure.recorded` | Analytics pipeline |
| `feature.schedule.applied` | Ops log |

---

## 20. Compatibility notes

- Public prefix `/api/v1/features`; schema `feature`.  
- Flag keys are immutable after publish; use aliases for renames.  
- `client_side_available` false flags never appear in bootstrap.  
- AND licenses explicitly in context or in API guards — do not assume flag means entitled.

---

## 21. Related documents

- Guide: [`FEATURE_GUIDE.md`](FEATURE_GUIDE.md)  
- Schema: [`FEATURE_SCHEMA.md`](FEATURE_SCHEMA.md)  
- Configuration: [`../03_configuration/CONFIGURATION_API.md`](../03_configuration/CONFIGURATION_API.md)  
- Rules: [`../11_rules/RULES_API.md`](../11_rules/RULES_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
