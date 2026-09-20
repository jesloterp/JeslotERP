# JeslotERP Feature Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — flag/override HTTP persists on Postgres; empty catalog/kills/overrides is `[]`. Evaluate stays CatalogStore. License compile with p26 later. Not Production.  
**Package:** `platforms.p12_feature`  
**PostgreSQL schema:** `feature`  
**Depends on:** `p01_identity`, `p02_organization`, `p03_configuration`  
**Integrates with:** `p05_metadata` (UI variants), `p06_localization`, `p10_process`, `p11_rules` (context facts), `p16_cache`, `p17_scheduler` (scheduled flips), `p26_licensing` (entitlement gates), `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`FEATURE_SCHEMA.md`](FEATURE_SCHEMA.md) · [`FEATURE_API.md`](FEATURE_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise feature-flag plane: flags/variants, environments, segments, targeting, % rollout, sticky buckets, prerequisites, kill switches, experiments, scheduled changes, overlays, evaluate/explain, packs. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-014: flag/kill/override HTTP dual-write + DB-first list; empty list is `[]`; `require_feature_access` sets RLS GUCs. Evaluate/bootstrap remain CatalogStore. p26 license compile deferred. |

---

## 1. Purpose (enterprise)

`p12_feature` is JeslotERP’s **feature flag & staged rollout control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **LaunchDarkly / Unleash / Flagsmith** — targeting, percentage rollout, segments, kill switches  
- **Salesforce** — Feature Management / permission-set driven exposure patterns  
- **Microsoft Dynamics 365** — feature management / flighting for gradual enablement  
- **SAP** — Switch Framework / business function activation (enterprise toggle discipline)

It is **not** a boolean in `.env`. It is the system that makes ERP releases safe for:

1. **Gradual rollout** by tenant, company, branch, user, role, plan  
2. **Kill switches** for incident response without redeploy  
3. **Multivariate flags** (theme/layout/pricing path variants)  
4. **Segments** reusable across flags  
5. **Percentage / sticky bucketing** for consistent UX  
6. **Prerequisites** (flag B only if flag A on)  
7. **Environment isolation** (dev/stage/prod definitions)  
8. **Scheduled enable/disable** windows  
9. **Experiments** (lite A/B with exposure logging)  
10. **Evaluate + explain** for support/debug and rules/process context  

### Owns

| Domain | Examples |
|---|---|
| Flag catalog | keys, types, owners, tags |
| Environments | development, staging, production |
| Targeting rules | if/then serve variation |
| Segments | reusable audiences |
| Rollouts | % progressive delivery |
| Bucketing | sticky hash assignments |
| Prerequisites | dependency graph |
| Kill switches | emergency off |
| Experiments | exposure & variants |
| Schedules | future flips |
| Evaluation runtime | effective flag state |
| Governance | publish, approvals, packs |

### Does **not** own

| Concern | Owner |
|---|---|
| Long-lived business settings | `p03_configuration` |
| Entitlements / paid SKUs | `p26_licensing` (may feed targeting) |
| IAM permissions | `p01_identity` (role is a target attribute) |
| Decision tables | `p11_rules` (consumes flag facts) |
| UI field catalogs | `p05_metadata` (may key off flags) |

### Critical split: Feature vs Configuration vs Licensing

| | **Feature (p12)** | **Configuration (p03)** | **Licensing (p26)** |
|---|---|---|---|
| Cadence | Release / experiment | Business preference | Commercial contract |
| Example | `sales.order.bulk_import.v2` | `bilty.default_payment_term` | `module.sales.advanced` |
| Default | Off until rolled out | Tenant-defined value | Entitled or not |

**Rule:** Flags gate *capability exposure*. Settings store *values*. Licenses gate *commercial right* (often AND-ed with flags).

---

## 2. Architectural position

```text
Client / API / Worker
        │
        ▼
  evaluate(flag_keys, context)
        │
        ├── environment
        ├── targeting rules / segments
        ├── % rollout + sticky bucket
        ├── prerequisites
        └── license/config fact hooks (optional)
        │
        ▼
  { value, variation, reason, explain }
```

**Hard rules**

1. Evaluation is **read-mostly** and cacheable; changes invalidate by env+flag.  
2. No cross-schema FKs — context carries UUIDs.  
3. Production changes are audited; kill switch is privileged + fast path.  
4. RLS fail-closed on tenant overrides.  
5. Never require redeploy to disable a broken feature path.

---

## 3. Advanced design principles

1. **Flag-key stability** — `domain.capability.version` naming (`sales.order.bulk_import.v2`).  
2. **Typed flags** — BOOLEAN, STRING, NUMBER, JSON, VARIANT.  
3. **Environment-scoped delivery** — same key, different rules per env.  
4. **Fallthrough + off variation** — always defined defaults.  
5. **Rule order FIRST-match** for targeting (like LD).  
6. **Segments as first-class** audiences.  
7. **Sticky bucketing** — `hash(flag_key + bucket_key)` for stable %.  
8. **Prerequisites** — short-circuit OFF if dependency OFF.  
9. **Kill switch** — forces OFF ignoring targeting (except break-glass).  
10. **Explain reasons** — `TARGET_MATCH`, `PERCENT_ROLLOUT`, `FALLTHROUGH`, `KILL_SWITCH`, `PREREQ_FAILED`, `TENANT_OVERRIDE`, …  
11. **Client bootstrapping** — bulk evaluate for SPA shell.  
12. **Server authoritative** — browser flags are hints; server re-checks.  
13. **Change scheduling** — future activate with scheduler worker.  
14. **Experiment exposures** — log once per subject/flag for analytics.  
15. **Idempotent upsert** of rules; publish immutability for rule sets optional.  
16. **CQRS HTTP** — thin routers; evaluator in application layer.  
17. **Cache + ETag** on bootstrap payloads.  
18. **Packs** — module feature packs seeded on install.

---

## 4. Core concepts

### 4.1 Flag

```text
flag_key, flag_type, description, owner,
variations[], defaults (on/off/fallthrough),
temporary? (debt marker), tags
```

### 4.2 Environment config

Per env: rules[], fallthrough, off_variation, on/off kill, prerequisites, percent rollout salt.

### 4.3 Context (evaluation)

```text
env,
tenant_id, company_id?, branch_id?,
user_id?, roles[],
plan_code?, license_modules[],
attributes{}   # custom: region, beta_tester, …
bucket_key     # default user_id or tenant_id
```

### 4.4 Targeting rule

```text
if segment OR clauses (attr op value) → serve variation
clauses: EQ, NEQ, IN, NOT_IN, CONTAINS, STARTS_WITH, SEMVER_*, NUMBER_*
```

### 4.5 Percentage rollout

Serve variation to `p%` of bucket_key space; sticky across sessions.

### 4.6 Tenant / company overrides

Explicit FORCE_ON / FORCE_OFF / FORCE_VARIATION — highest priority after kill switch (policy order documented).

### 4.7 Effective precedence (default)

```text
1. KILL_SWITCH (global off)
2. ENV disabled
3. PREREQUISITE failed
4. TENANT/COMPANY override
5. Targeting rules (first match)
6. Percentage rollout (if configured on fallthrough)
7. FALLTHROUGH variation
8. OFF variation if flag archived/inactive
```

---

## 5. Experiments (lite)

- Experiment binds flag + hypothesis + start/end  
- Exposure event when subject first evaluated into non-off variation  
- No full stats engine in p12 — export events to analytics  
- Guardrails: max concurrent experiments per tenant optional  

---

## 6. Integration patterns

| Consumer | Use |
|---|---|
| API guards | Skip/enable route behavior |
| Metadata UI packs | Channel/feature variants |
| Process/rules | Context provider `feature.flags` |
| Frontend shell | Bootstrap bundle with ETag |
| Workers | Evaluate with service context |

**Licensing AND:** `entitled(module) && flag_on(capability)`.

---

## 7. Security

### Permissions

| Code | Use |
|---|---|
| `feature.catalog.read` | Read flags |
| `feature.catalog.manage` | Create/edit flags |
| `feature.targeting.manage` | Rules/segments |
| `feature.override.manage` | Tenant overrides |
| `feature.kill` | Kill switch |
| `feature.publish` | Activate scheduled/env promote |
| `feature.evaluate` | Runtime evaluate (usually authenticated users) |
| `feature.experiment.manage` | Experiments |
| `feature.pack.install` | Packs |
| `feature.audit.read` | Audit |
| `feature.*` | Wildcard |

### RLS

FORCE RLS on tenant overrides, exposure logs (if tenant-scoped), segments with tenant scope.

---

## 8. Module layout

```text
platforms/p12_feature/
  application/
    services/
      evaluator.py
      targeting.py
      bucketing.py
      segment_matcher.py
      prerequisite.py
      kill_switch.py
      bootstrap.py
      scheduler_apply.py
      experiment_exposure.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… messaging/outbox/ cache/
  tests/unit/evaluator/ bucketing/ targeting/
```

---

## 9. Domain events

| Event | When |
|---|---|
| `feature.flag.created` / `updated` / `archived` | Catalog |
| `feature.env.rules.changed` | Targeting |
| `feature.kill.engaged` / `cleared` | Incidents |
| `feature.override.changed` | Tenant ops |
| `feature.schedule.applied` | Scheduler |
| `feature.experiment.started` / `stopped` | Experiments |
| `feature.exposure.recorded` | Analytics |

Stream: `jesloterp:feature:outbox`.

---

## 10. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, envs, permissions |
| P2 | Boolean flags + evaluate |
| P3 | Targeting rules + segments |
| P4 | % rollout + sticky buckets |
| P5 | Prerequisites + kill switch |
| P6 | Variants + bootstrap ETag |
| P7 | Overrides + schedules |
| P8 | Experiments + exposures |
| P9 | Packs + promote-across-env |
| P10 | Registry → **SoR-Live** (flag/override HTTP on Postgres; evaluate still CatalogStore) |

---

## 11. Definition of Done (enterprise)

- [x] Sticky bucket stability tests  
- [x] Precedence order unit tests  
- [x] Kill switch bypasses targeting  
- [x] Prerequisite short-circuit  
- [x] Bootstrap ETag 304  
- [x] Tenant override RLS  
- [x] Flag/override HTTP Postgres-first; empty list is `[]`  
- [x] Explain reason codes covered  
- [x] Server re-check documented for client flags  
- [x] No cross-schema FKs  

---

## 12. Anti-patterns

| Don’t | Do |
|---|---|
| Config setting as release flag | Use p12 flag |
| Hardcode tenant allowlists in code | Targeting / segments |
| Trust only browser flag cache | Re-evaluate on server |
| Leave permanent flags forever | Mark temporary + cleanup debt |
| % rollout without sticky key | Explicit bucket_key |
| Entitlement only via flag | License AND flag |

---

## 13. Related documents

- Schema: [`FEATURE_SCHEMA.md`](FEATURE_SCHEMA.md)  
- API: [`FEATURE_API.md`](FEATURE_API.md)  
- Configuration: [`../03_configuration/CONFIGURATION_GUIDE.md`](../03_configuration/CONFIGURATION_GUIDE.md)  
- Licensing: [`../26_licensing/LICENSING_GUIDE.md`](../26_licensing/LICENSING_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
