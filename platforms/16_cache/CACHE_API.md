# JeslotERP Cache Platform — Complete API Specification (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — namespace catalog Postgres-first; Redis `test-connection` is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p16_cache`  
**PostgreSQL schema:** `cache`  
**Public base:** `/api/v1/cache`  
**Internal base:** `/internal/v1/cache`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`CACHE_GUIDE.md`](CACHE_GUIDE.md) · [`CACHE_SCHEMA.md`](CACHE_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Namespaces/policies, key simulate, invalidate/purge, warmup, stats, backends, internal SDK ops, packs. |
| 1.2 | 2026-09-12 | TASK-SOR-026: `POST /backends/{key}/test-connection`; Redis never invented healthy; empty namespace list is `[]`. |

---

## 1. Design principles (advanced)

1. **Control-plane HTTP** — humans/ops invalidate & configure; hot get/set is **internal SDK**.  
2. **No end-user arbitrary key read/write API** — prevents cache as data exfil channel.  
3. **Tag invalidate preferred** over pattern scans.  
4. **Idempotent invalidate**.  
5. **Production purge gated** — approval + confirm token.  
6. **Tenant scoped** invalidations default to caller tenant.  
7. **Explain key build** — simulate templates.  
8. **Metrics first-class**.  
9. **Backend secrets never returned**.  
10. **Invalidate bus** reliable via outbox.  
11. **Warmup allow-listed loaders only**.  
12. **Environment isolation** enforced on backend binding.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Cache-Env: production
X-Internal-Token: <token>
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": { "request_id": "…" }
}
```

---

## 4. Errors

```text
CCH_NAMESPACE_NOT_FOUND / TEMPLATE_NOT_FOUND / BACKEND_NOT_FOUND
CCH_KEY_INVALID / TENANT_REQUIRED / VALUE_TOO_LARGE
CCH_SENSITIVITY_FORBIDDEN
CCH_INVALIDATE_FAILED / PURGE_DENIED / APPROVAL_REQUIRED
CCH_WARMUP_LOADER_UNKNOWN / WARMUP_FAILED
CCH_QUOTA_EXCEEDED / RATE_LIMITED
CCH_BACKEND_UNHEALTHY
CCH_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
CCH_PACKAGE_CHECKSUM_MISMATCH
CCH_SDK_UNAUTHORIZED
CCH_CONFIRM_REQUIRED
```

HTTP: `404` · `409` · `422` · `403` · `429` · `503`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `cache.catalog.read` | Read namespaces/policies |
| `cache.catalog.manage` | Mutate catalog |
| `cache.invalidate` | Invalidate |
| `cache.purge` | Broad purge |
| `cache.warmup` | Warmup |
| `cache.stats.read` | Metrics |
| `cache.backend.manage` | Backends |
| `cache.pack.install` | Packs |
| `cache.audit.read` | Audit |
| `cache.*` | All |

---

## 6. Internal SDK-equivalent APIs (services only)

> Used by platform workers / API nodes. Not for browsers.

### 6.1 Get / Set / Delete

```http
POST /internal/v1/cache/get
POST /internal/v1/cache/set
POST /internal/v1/cache/delete
POST /internal/v1/cache/get-or-load
```

**Get:**

```json
{
  "namespace_key": "rules.compiled",
  "key": "rules:tenant:…:finance.credit.requires_approval:v3",
  "layers": ["L1_MEMORY", "L2_REDIS"]
}
```

**Set:**

```json
{
  "namespace_key": "rules.compiled",
  "key": "…",
  "value": { "…": "compiled blob" },
  "ttl_sec": 3600,
  "tags": ["tenant:…", "rule:finance.credit.requires_approval"],
  "soft_ttl_sec": 3000
}
```

**Get-or-load:**

```json
{
  "namespace_key": "rules.compiled",
  "key": "…",
  "tags": ["tenant:…", "rule:…"],
  "loader_key": "rules.load_compiled",
  "loader_args": { "rule_key": "finance.credit.requires_approval", "version": 3 },
  "single_flight": true
}
```

Loader executed only on miss; must be allow-listed server-side (not arbitrary code).

### 6.2 Build key

```http
POST /internal/v1/cache/keys/build
POST /api/v1/cache/keys/simulate
```

```json
{
  "namespace_key": "metadata.effective",
  "template_key": "entity_effective",
  "params": {
    "tenant_id": "…",
    "entity_key": "bp.Partner",
    "version": 12
  }
}
```

Returns fully qualified key + validation errors if any.

---

## 7. Invalidation APIs (primary ops)

### 7.1 Invalidate

```http
POST /api/v1/cache/invalidate
Idempotency-Key: …
```

```json
{
  "reason": "rules.definition.activated",
  "items": [
    { "type": "TAG", "value": "rule:finance.credit.requires_approval" },
    { "type": "TAG", "value": "tenant:…" },
    { "type": "KEY", "namespace_key": "feature.env", "value": "feature:env:production:sales.order.bulk_import.v2" }
  ]
}
```

**Response:** `invalidate_request_id`, status, estimated keys (if known).

Propagates to L2 and publish invalidate bus for L1.

### 7.2 Invalidate by event helper

```http
POST /internal/v1/cache/invalidate-for-event
```

Maps known domain events → tag sets (from pack rules).

### 7.3 Get invalidate status

```http
GET /api/v1/cache/invalidate/{request_id}
```

---

## 8. Purge (dangerous)

```http
POST /api/v1/cache/purge
```

```json
{
  "scope": "NAMESPACE",
  "namespace_key": "i18n.bundle",
  "env": "production",
  "confirm_token": "PURGE namespace=i18n.bundle env=production",
  "changeset_id": "…"
}
```

`scope=BACKEND` / whole-env requires `cache.purge` + approval.  
`ALL_ENV_FORBIDDEN` always rejected.

---

## 9. Warmup

```http
GET  /api/v1/cache/warmup/plans
POST /api/v1/cache/warmup/plans
POST /api/v1/cache/warmup/plans/{plan_key}/run
GET  /api/v1/cache/warmup/runs/{id}
```

**Run:**

```json
{
  "tenant_id": "…",
  "params": { "locales": ["en", "hi-IN"] }
}
```

Enqueues p14 jobs per item; loaders allow-listed.

---

## 10. Catalog — namespaces & policies

```http
GET    /api/v1/cache/namespaces
POST   /api/v1/cache/namespaces
GET    /api/v1/cache/namespaces/{namespace_key}
PATCH  /api/v1/cache/namespaces/{namespace_key}
PUT    /api/v1/cache/namespaces/{namespace_key}/policies
GET    /api/v1/cache/namespaces/{namespace_key}/templates
PUT    /api/v1/cache/namespaces/{namespace_key}/templates/{template_key}

GET    /api/v1/cache/policies/ttl
POST   /api/v1/cache/policies/ttl
GET    /api/v1/cache/tags/defs
```

---

## 11. Backends & health

```http
GET  /api/v1/cache/backends
POST /api/v1/cache/backends
PATCH /api/v1/cache/backends/{backend_key}
POST /api/v1/cache/backends/{backend_key}/test
POST /api/v1/cache/backends/{backend_key}/test-connection
GET  /api/v1/cache/backends/{backend_key}/health
PUT  /api/v1/cache/namespaces/{namespace_key}/backend
```

`memory_l1` / MEMORY → `ok: true`. `redis_primary` / REDIS → `ok: false`, `status: PROVIDER_PENDING` unless a live Redis L2 is attached and ping succeeds. Never echoes secrets. Requires `cache.backend.manage` via `require_cache_access`.

Namespace list/get persist on `AsyncSession`; empty catalog is `[]`. `buffer_class=NONE` on create refuses later `cache_set`.

---

## 12. Quotas

```http
GET /api/v1/cache/quotas
PUT /api/v1/cache/quotas/tenants/{tenant_id}
GET /api/v1/cache/quotas/tenants/{tenant_id}/usage
```

---

## 13. Stats & alerts

```http
GET /api/v1/cache/stats?namespace_key=rules.compiled&from=…&to=…
GET /api/v1/cache/stats/hit-ratio
GET /api/v1/cache/slow-keys?namespace_key=…
GET /api/v1/cache/alerts
PUT /api/v1/cache/alert-rules/{rule_key}
```

---

## 14. Packages & governance

```http
GET  /api/v1/cache/packages
POST /api/v1/cache/packages/{package_key}/install
GET  /api/v1/cache/changesets
POST /api/v1/cache/changesets
POST /api/v1/cache/changesets/{id}/approvals
```

---

## 15. Audit

```http
GET /api/v1/cache/audit/invalidations?from=…&to=…
GET /api/v1/cache/audit/purges
```

Requires `cache.audit.read`.

---

## 16. Invalidation bus (internal)

```http
POST /internal/v1/cache/bus/publish
POST /internal/v1/cache/bus/tick
POST /internal/v1/cache/l1/drop
```

- Nodes call `l1/drop` locally on receipt  
- `bus/tick` relays outbox reliably  

**Drop body:**

```json
{
  "keys": ["…"],
  "tags": ["tenant:…"],
  "namespaces": ["rules.compiled"]
}
```

---

## 17. Health

```http
GET /api/v1/cache/health
GET /internal/v1/cache/health
```

Reports backend ping, invalidate lag, L1 bus consumers.

---

## 18. Caching & concurrency (meta)

| Resource | Strategy |
|---|---|
| Namespace policies | Cached in L1 with short TTL + invalidate on change |
| Invalidate | Fan-out async; request row tracks completion |
| Single-flight | Redis lock key with TTL |
| Tag membership | Redis SET primary |

---

## 19. Example flows

### 19.1 Rules activate

1. p11 activates definition  
2. Emits event → `invalidate-for-event`  
3. Tags `rule:{key}` purged L2 + L1 bus  
4. Next evaluate get-or-load recompiles  

### 19.2 i18n publish

1. Invalidate tag `locale:hi-IN` + namespace `i18n.bundle` keys for tenant  
2. Warmup plan runs for active locales  

### 19.3 Incident stampede

1. Soft TTL serves stale  
2. Single-flight loader refreshes once  
3. Metrics show miss spike without DB melt  

### 19.4 Bad deploy cache poison

1. Pause writers if needed  
2. Approved purge namespace  
3. Warmup critical plans  
4. Resume  

---

## 20. Event hooks

| Event | Consumer |
|---|---|
| `cache.invalidate.requested` | Nodes / workers |
| `cache.backend.unhealthy` | On-call |
| `cache.quota.breached` | Tenant admins |
| `cache.warmup.completed` | Release pipeline |

---

## 21. Compatibility notes

- Public prefix `/api/v1/cache`; schema `cache`.  
- Platforms should depend on shared SDK, not raw Redis clients.  
- Browser never receives internal get/set.  
- ETags on HTTP APIs remain platform-owned; they may store digests in p16.

---

## 22. Related documents

- Guide: [`CACHE_GUIDE.md`](CACHE_GUIDE.md)  
- Schema: [`CACHE_SCHEMA.md`](CACHE_SCHEMA.md)  
- Configuration: [`../03_configuration/CONFIGURATION_API.md`](../03_configuration/CONFIGURATION_API.md)  
- Messaging: [`../14_messaging/MESSAGING_API.md`](../14_messaging/MESSAGING_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
