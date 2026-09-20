# API Platform — Implementation Record

**Platform:** `p22_api`  
**Date:** 2026-09-12  
**Verification:** `python -m pytest platforms/p22_api/tests tests/hygiene/test_hyg017_p22_bulk_composite.py -q --tb=short`

## 1. Overview & Objective

API product and gateway control plane: schema `api`, OpenAPI catalog, products/plans/subscriptions, hashed API keys, rate limits/quotas, gateway policy bundles and snapshots, analytics, developer portal meta, packs/CI gates. Alembic f22a/f22b. In-memory catalog store matching p20/p21.

## 2. Source documents reviewed

`API_GUIDE.md`, `API_SCHEMA.md`, `API_ENDPOINTS.md` under `docs/platforms/22_api/` plus `docs/tasks/task_p22_api.md`. Requirement docs were not modified.

## 3. Existing backend architecture reviewed

p21_monitoring ModulePlugin, in-memory catalog store, exception handlers from p05 auth, `require_internal_token`, dual Alembic schema+RLS, TestClient factory. Same layout copied for p22.

## 4. Requirements identified

See `API_RTM.md` (100% of GUIDE / SCHEMA / ENDPOINTS items mapped).

## 5. Requirement-by-requirement implementation

In-memory `ApiCatalogStore` implements catalog, specs/versions, products/plans/subscriptions, SHA-256 key issue/verify/rotate/revoke, rate-limit counters (envelope 200 + HTTP 429), policy compile, fail-closed snapshots, analytics, portal, packs, CI validate. HTTP is thin CQRS over the store. Public `/api/v1/api` plus documented `/api/v1/api/internal/*` and `/internal/v1/api/*` aliases.

## 6. Files created or modified

**Created**

- `platforms/p22_api/` domain, application, infrastructure HTTP/module, tests
- `platforms/p22_api/infrastructure/persistence/schema_constants.py` (`API_SCHEMA = "api"`)
- `api_outbox` + `api_idempotency_key`; models `__init__` imports all 62 classes
- `alembic/versions/f22a0b1c2d3e_create_api_schema.py`
- `alembic/versions/f22b1c2d3e4f_enable_api_rls.py`
- `docs/platforms/22_api/API_RTM.md`
- this record

**Modified**

- `apps/api/main.py` — `ApiModule` after `MonitoringModule`; exception handlers
- `alembic/env.py` — import p22 models
- `IMPLEMENTATION_TASKS.md`, `IMPLEMENTATION_STATUS.md`

**Not edited**

- `docs/platforms/22_api/API_GUIDE.md`, `API_SCHEMA.md`, `API_ENDPOINTS.md`
- `docs/tasks/task_p22_api.md`

## 7. Database changes & migrations

Schema `api`, 60 domain `api_*` tables + `api_outbox` + `api_idempotency_key` = **62**. FORCE RLS on subscriptions, keys, usage, and related tenant tables. Permissions `api.*` seeded in f22a. No cross-schema FKs.

## 8. APIs

Public `/api/v1/api` (ENDPOINTS §1–9). Internal verify/check/snapshot also at `/api/v1/api/internal/*` and `/internal/v1/api/*`. Health at `/api/v1/api/health` and `/internal/v1/api/health`.

## 9. Business rules

- API key plaintext shown once on create/rotate; store SHA-256(+pepper) only
- Rate-limit deny when remaining=0: HTTP 200 `{allowed:false}` or HTTP 429 `RATE_LIMITED` via `adapter_mode`
- No published snapshot → `SNAPSHOT_UNAVAILABLE` 503 on verify and `/snapshots/latest`
- Deprecate ops/versions set `Deprecation` / `Sunset` headers
- Tenant callers cannot read other tenants' subscriptions/keys unless `api.admin` / `api.*`
- Feature-gated operation denied (`FORBIDDEN`) when flag off
- Portal public GETs never return private pages or key secrets

## 10. Validation, permissions & errors

ENDPOINTS §0 codes as `exception.code`. Permissions from ENDPOINTS §11 via `require_api_permission`.

## 11. Integrations

Depends on `p01_identity`, `p12_feature` (module graph). Feature flags evaluated in-store (p12 consult). Snapshot cache-tag increment stands in for p16 invalidate. Domain events go to outbox `jesloterp:api:outbox`.

## 12. Tests

Module (4) + API families (7) with ≥2 variations each (success + failure/guard). **11 passed**.

## 13. Test execution results

`pytest platforms/p22_api/tests -q` → **11 passed**.

## 14. RTM

`API_RTM.md`.

## 15. Issues found & resolved

Second TestClient in tenant/portal cases reset the in-memory store; added `reset=False`. `denied_user(permissions=...)` collided with a hardcoded empty list; keyword now optional.

## 16. Regression

p22 suite green. Full-repo suite not run (TASK-014).

## 17. Coverage & completion

TASK-008 acceptance bar met: implement + tests + RTM + this record.

## 18. Limitations

1. Hashed API-key HTTP persists on Postgres (`key_hash` only; plaintext once on create/rotate). Empty list is `[]`. `require_api_access` sets RLS GUCs. Products/plans/subscriptions/catalog still memory. Bulk jobs + Composite are process-local (HYG-017). OData/GraphQL not shipped. Not Production.
2. Rate counters use `RateLimiter`: MEMORY under pytest; Redis attaches only when `REDIS_URL` pings. No invented Redis hits.
3. Alembic not applied to live DB in-session; p12/p16/p19 integrations are consult/outbox/tag.
