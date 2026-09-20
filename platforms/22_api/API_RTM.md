# API Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p22_api/tests tests/hygiene/test_hyg017_p22_bulk_composite.py -q --tb=short`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| API-G-01 | GUIDE §1 | API product & gateway control plane | `ApiCatalogStore` + schema `api` | Implemented | module tables + health | PASS |
| API-G-02 | GUIDE §2 | Production routes registered in catalog (CI) | `register_batch` + `/ci/gates/validate` | Implemented | catalog batch + CI pass/fail | PASS |
| API-G-03 | GUIDE §2 | API keys hashed; plaintext once | `create_key` / `rotate_key` / SHA-256 | Implemented | keys hash/rotate/list | PASS |
| API-G-04 | GUIDE §2 | Rate limit fast; PG holds policy | in-memory counters + `api_rate_policy` | Implemented | rate allow + both deny paths | PASS |
| API-G-05 | GUIDE §2 | Deprecation Sunset / Deprecation headers | `deprecate_operation` / `deprecate_version` | Implemented | catalog deprecate + version deprecate | PASS |
| API-G-06 | GUIDE §2 | No cross-schema FKs | UUID/string refs only on ORM | Implemented | table inventory | PASS |
| API-G-07 | GUIDE §2 | RLS on tenant subscriptions & keys | Alembic `f22b1c2d3e4f` | Implemented | migration present + tenant isolation | PASS |
| API-G-08 | GUIDE §3 | Operation-first catalog | POST services/operations | Implemented | catalog family | PASS |
| API-G-09 | GUIDE §3 | OpenAPI checksummed specs | spec versions + publish | Implemented | specs publish checksum | PASS |
| API-G-10 | GUIDE §3 | Products / plans / subscriptions | product/plan/sub APIs | Implemented | products family | PASS |
| API-G-11 | GUIDE §3 | Keys bind to subscription | create/list/revoke/rotate | Implemented | keys family | PASS |
| API-G-12 | GUIDE §3 | Multi-dimensional limits | plan limits + rate policies | Implemented | limits + invalid dimension | PASS |
| API-G-13 | GUIDE §3 | Idempotency / CORS / IP policies | policy APIs | Implemented | policies family | PASS |
| API-G-14 | GUIDE §3 | Revision vs version lifecycle | `/versions` activate/deprecate | Implemented | versions family | PASS |
| API-G-15 | GUIDE §3 | Analytics sampled meta | `/analytics/*` | Implemented | analytics family | PASS |
| API-G-16 | GUIDE §3 | Packs seed core public product | `core.public.v1` apply | Implemented | packages apply | PASS |
| API-G-17 | GUIDE §6 | Permissions `api.*` | HTTP gates | Implemented | 403 catalog | PASS |
| API-G-18 | GUIDE §8 | Domain events spec/key/sub/limit/snapshot/product | outbox `_emit` | Implemented | keys/subs/snapshots emit in store | PASS |
| API-G-19 | GUIDE §10 | DoD: key hash, 429+Retry-After, sunset, snapshot fail-closed, tenant isolation, feature flag, no XFKs | store + APIs | Implemented | keys/rate/snapshot/tenant/flag | PASS |
| API-S-01 | SCHEMA §2 | 60 domain + plumbing = 62 `api_*` | ORM + outbox + idempotency | Implemented | `test_api_module_tables_count_62` | PASS |
| API-S-02 | SCHEMA §1 | Schema `api`, secrets hash-only | `API_SCHEMA` + key_hash | Implemented | tables + keys | PASS |
| API-S-03 | SCHEMA §3 | Enums method/lifecycle/auth/limit/key/sub/deprecation | `domain/enums.py` | Implemented | API payloads | PASS |
| API-S-04 | SCHEMA §4–10 | Catalog/spec/product/key/limit/policy/analytics columns | ORM models | Implemented | table inventory | PASS |
| API-S-05 | SCHEMA §12 | `api_outbox`, `api_idempotency_key` | outbox + idempotency models | Implemented | table names + stream | PASS |
| API-S-06 | SCHEMA §13 | FORCE RLS tenant subscriptions/keys/usage | Alembic `f22b1c2d3e4f` | Implemented | migration present | PASS |
| API-S-07 | SCHEMA §14 | Seed v1, products, plans, idempotency, CORS, perms, errors | `seed_defaults` + Alembic perms | Implemented | list services/products/versions | PASS |
| API-S-08 | SCHEMA §16 | SHA-256+pepper verify; deterministic snapshot checksum | `_hash_key` / `_checksum` | Implemented | verify + publish | PASS |
| API-E-00 | ENDPOINTS §0 | Envelope + error codes AUTH/FORBIDDEN/NOT_FOUND/CONFLICT/VALIDATION/RATE_LIMITED/SNAPSHOT_UNAVAILABLE | `domain/exceptions.py` | Implemented | all API families | PASS |
| API-E-01 | ENDPOINTS §1 | Catalog services/ops/deprecate/register-batch | `/api/v1/api/services*` `/operations*` `/catalog/register-batch` | Implemented | catalog family | PASS |
| API-E-02 | ENDPOINTS §2 | Specs/versions/changelog | `/specs` `/versions` | Implemented | specs family | PASS |
| API-E-03 | ENDPOINTS §3 | Products/plans/limits/subscriptions | `/products` `/plans` `/subscriptions` | Implemented | products family | PASS |
| API-E-04 | ENDPOINTS §4 | Keys hash+revoke+rotate, oauth, verify | `/keys` `/oauth-bindings` `/internal/keys/verify` | Implemented | keys family | PASS |
| API-E-05 | ENDPOINTS §5 | Rate policies/check (200 allowed false or 429), bundles/CORS/IP/idempotency | `/rate-policies` `/internal/rate-limit/check` | Implemented | rate + policies | PASS |
| API-E-06 | ENDPOINTS §6 | Snapshots publish/latest/internal get | `/snapshots/*` | Implemented | snapshots family | PASS |
| API-E-07 | ENDPOINTS §7 | Analytics usage/top-routes/errors/latency | `/analytics/*` | Implemented | analytics family | PASS |
| API-E-08 | ENDPOINTS §8 | Portal pages/sdk/tryit; public never leaks keys | `/portal/*` | Implemented | portal public vs private | PASS |
| API-E-09 | ENDPOINTS §9 | Packages apply, CI gates, changesets approve | `/packages` `/ci/gates` `/changesets` | Implemented | gov family | PASS |
| API-E-10 | ENDPOINTS §0/4/5 | Internal aliases `/internal/v1/api/*` + health | internal router | Implemented | verify/check/snapshot/health aliases | PASS |
| API-E-11 | ENDPOINTS §11 | Permission matrix | `require_api_permission` | Implemented | 403 + tenant RLS | PASS |
| API-MOD | registry / brief | ModulePlugin after p21; deps p01+p12; Alembic f22a/f22b | `ApiModule` + main/env | Implemented | load order + deps | PASS |
| API-SOR-01 | TASK-SOR-020 | Hashed API keys → Postgres; empty `[]` | `credential_repository` + `require_api_access` | Implemented | persist-then-fetch + db-first empty | PASS |
| API-SOR-02 | TASK-SOR-020 | Redis rate-limit port; pytest MEMORY | `RedisRateLimiter` + `try_attach_redis_rate_limiter` | Implemented | redis limiter tests | PASS |
| API-HYG-017 | AUD-017 | Bulk + Composite on p22; no OData/GraphQL | `BatchGateway` + `/composite` `/bulk/jobs` | Implemented | `test_bulk_composite` + hygiene lock | PASS |
