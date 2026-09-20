# Cache Platform — Requirements Traceability Matrix (RTM)

**Platform:** `p16_cache` · **Schema:** `cache`  
**Sources:** CACHE_GUIDE.md · CACHE_SCHEMA.md · CACHE_API.md · docs/tasks/task_p16_cache.md  
**Verification date:** 2026-09-11  
**Test command:** `pytest platforms/p16_cache/tests -q --tb=short` → **44 passed**  
**Alembic:** `f16a0b1c2d3e` → `f16b1c2d3e4f` (down_revision from `f15b1c2d3e4f`)  
**ORM tables:** **61** (`58` domain + `cch_outbox` + `cch_idempotency_key` + `cch_catalog_audit`)

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| CCH-G-01 | GUIDE §1 | Caching control plane (namespaces, keys, TTL, tags, L1/L2) | `platforms/p16_cache` ModulePlugin | Done | `test_cch_module_*` | Passed |
| CCH-G-02 | GUIDE §1 | Governed namespaces (feature, i18n, metadata, rules, session) | seed + catalog APIs | Done | namespace list/create tests | Passed |
| CCH-G-03 | GUIDE §1 | Key standards — templates, tenant scoping, version suffixes | `key_builder` + templates | Done | keys unit + simulate API | Passed |
| CCH-G-04 | GUIDE §1 | TTL & tier policies | ttl policies + binds | Done | ttl create/list + policy put | Passed |
| CCH-G-05 | GUIDE §1 | Tag-based invalidation | tag_index + invalidator | Done | tag invalidate unit + API | Passed |
| CCH-G-06 | GUIDE §1 | Multi-layer L1 memory + L2 Redis stand-in | catalog_store l1/l2 | Done | get/set layer + L1 drop | Passed |
| CCH-G-07 | GUIDE §1 | Stampede protection single-flight / soft TTL | `get_or_load` + stampede policy | Done | single_flight unit + internal get-or-load | Passed |
| CCH-G-08 | GUIDE §1 | Warmup & prefetch after publish | warmup plans/runs | Done | warmup run + unknown loader | Passed |
| CCH-G-09 | GUIDE §1 | Tenant isolation & quotas | quotas APIs + RLS | Done | quotas put/usage + RLS migration | Passed |
| CCH-G-10 | GUIDE §1 | Sensitive entry encryption refs (no plaintext secrets) | sensitivity policy + secret_ref_key | Done | forbidden set + backend no-echo | Passed |
| CCH-G-11 | GUIDE §1 | Hit/miss/latency metrics + invalidation audit | metrics + audit APIs | Done | stats/hit-ratio + audit tests | Passed |
| CCH-G-12 | GUIDE §2 | Cache is ephemeral; SoT stays in domain DB | design + no value in PG meta | Done | key_meta model has no value col | Passed |
| CCH-G-13 | GUIDE §2 | Keys include namespace + tenant when scoped | template `require_tenant_param` | Done | `test_build_key_rejects_unscoped_*` | Passed |
| CCH-G-14 | GUIDE §2 | No unbounded KEYS * — tag index | tag_index dict / cch_tag_member | Done | tag invalidate removes members | Passed |
| CCH-G-15 | GUIDE §2 | Explicit invalidate on publish/activate | `invalidate-for-event` | Done | event map + unmapped | Passed |
| CCH-G-16 | GUIDE §2 | Secrets/tokens: encrypt-at-rest policy or don’t cache | FORBIDDEN sensitivity | Done | SensitivityForbiddenError test | Passed |
| CCH-G-17 | GUIDE §2 | RLS/authz on loader — cache not authz bypass | internal-only get/set + perms | Done | denied user + internal token | Passed |
| CCH-G-18 | GUIDE §2 | No cross-schema FKs — UUID refs only | ORM models | Done | model review | Passed |
| CCH-G-19 | GUIDE §3 | Namespace-first policies | namespace_policy_bind | Done | put policies API | Passed |
| CCH-G-20 | GUIDE §3 | Soft TTL + hard TTL | ttl_sec / soft_ttl_sec on set | Done | set response + stampede policy | Passed |
| CCH-G-21 | GUIDE §3 | Allow-listed loaders only | `ALLOWLISTED_LOADERS` | Done | unknown loader 422 | Passed |
| CCH-G-22 | GUIDE §3 | Size limits reject oversized values | `_max_bytes` | Done | `CCH_VALUE_TOO_LARGE` | Passed |
| CCH-G-23 | GUIDE §3 | Invalidate via pub/sub + outbox | outbox + bus tick/publish | Done | bus publish/tick/l1 drop | Passed |
| CCH-G-24 | GUIDE §3 | Environment separation | backend env_key isolation | Done | env mismatch 422 | Passed |
| CCH-G-25 | GUIDE §3 | Idempotent invalidate | Idempotency-Key | Done | same/conflict tests | Passed |
| CCH-G-26 | GUIDE §3 | Admin purge guarded in production | confirm_token + approval | Done | confirm / ALL_ENV / approval | Passed |
| CCH-G-27 | GUIDE §3 | CQRS HTTP — hot path is SDK | `application/sdk` + internal APIs | Done | CacheSdk + internal get/set | Passed |
| CCH-G-28 | GUIDE §3 | Packs seed platform namespaces | `platform.cache.core` | Done | packages list/install | Passed |
| CCH-G-29 | GUIDE §6 | Permissions cache.* | catalog + require_cache_permission | Done | denied + scoped stats.read | Passed |
| CCH-G-30 | GUIDE §6 | Tenant-scoped invalidate/stats FORCE tenant_id | Alembic `f16b1c2d3e4f` | Done | migration present | Passed |
| CCH-G-31 | GUIDE §6 | No end-user arbitrary key R/W | public has no get/set | Done | public routers catalog-only | Passed |
| CCH-G-32 | GUIDE §8 | Domain events outbox | `cch_outbox` + stream | Done | stream constant + bus | Passed |
| CCH-G-33 | GUIDE §10 | DoD: tenant keys, tag invalidate, single-flight, L1 drop, purge confirm, hit ratio, no secrets in PG, no X-FK | services + tests | Done | suite + table count | Passed |
| CCH-S-01 | SCHEMA §2 | 58 domain + 3 plumbing = 61 tables | ORM models | Done | `test_cch_module_tables_count_61` | Passed |
| CCH-S-02 | SCHEMA §1 | Schema name `cache` never p16 | `CACHE_SCHEMA` | Done | module/tables tests | Passed |
| CCH-S-03 | SCHEMA §3 | Enums partition/layer/write/consistency/sensitivity/invalidate/eviction/purge | `domain/enums.py` | Done | used across store/tests | Passed |
| CCH-S-04 | SCHEMA §14 | Seed backends, layers, namespaces, TTL, tag defs, stampede, permissions | store.seed_defaults + Alembic | Done | list APIs + migration | Passed |
| CCH-S-05 | SCHEMA §16 | Split models catalog/policy/backend/tags/invalidate/warmup/governance/plumbing | model packages | Done | import + count | Passed |
| CCH-S-06 | SCHEMA §6 | Backend secrets via secret_ref_key only | `cch_backend.secret_ref_key` | Done | backends no-echo test | Passed |
| CCH-S-07 | SCHEMA §13 | RLS FORCE on tenant invalidate/warmup/quota/usage | `f16b1c2d3e4f` | Done | revision chain | Passed |
| CCH-A-01 | API §3 | StandardResponse envelope | `route_common.ok` | Done | all API tests | Passed |
| CCH-A-02 | API §4 | Error codes CCH_* | `domain/exceptions.py` + handlers | Done | 403/404/409/422 tests | Passed |
| CCH-A-03 | API §5 | Permission codes cache.* | catalog + require_cache_permission | Done | denied + scoped perm | Passed |
| CCH-A-04 | API §6 | Internal get/set/delete/get-or-load + keys/build | internal router | Done | internal SDK tests (≥2) | Passed |
| CCH-A-05 | API §6.2 | Public keys/simulate | catalog router | Done | simulate success + tenant | Passed |
| CCH-A-06 | API §7 | Invalidate + status + invalidate-for-event | invalidate + internal | Done | idempotent + event (≥2) | Passed |
| CCH-A-07 | API §8 | Purge gated confirm + approval; ALL_ENV_FORBIDDEN | purge API | Done | confirm/forbidden/approval (≥2) | Passed |
| CCH-A-08 | API §9 | Warmup plans CRUD/run/runs | warmup router | Done | run + unknown + missing (≥2) | Passed |
| CCH-A-09 | API §10 | Namespaces/policies/templates/ttl/tag defs | catalog router | Done | catalog group (≥2) | Passed |
| CCH-A-10 | API §11 | Backends CRUD/test/health + ns backend bind | backends router | Done | no-secret + env isolation (≥2) | Passed |
| CCH-A-11 | API §12 | Quotas list/put/usage | ops router | Done | put + usage (≥2) | Passed |
| CCH-A-12 | API §13 | Stats / hit-ratio / slow-keys / alerts | ops router | Done | stats + missing ns (≥2) | Passed |
| CCH-A-13 | API §14 | Packages install + changesets/approvals | ops router | Done | checksum + approve (≥2) | Passed |
| CCH-A-14 | API §15 | Audit invalidations + purges | ops router | Done | both audit lists (≥2) | Passed |
| CCH-A-15 | API §16 | Internal bus publish/tick + l1/drop | internal router | Done | publish/tick/drop (≥2) | Passed |
| CCH-A-16 | API §17 | Public + internal health | ops + internal | Done | both health endpoints | Passed |
| CCH-W-01 | Wiring | main.py CacheModule after Notification + handlers | `apps/api/main.py` | Done | `test_cch_app_loads_p16_after_p15` | Passed |
| CCH-W-02 | Wiring | alembic env import models | `alembic/env.py` | Done | import path present | Passed |
| CCH-W-03 | Wiring | Migrations after notification head | `f16a0b1c2d3e` → `f16b1c2d3e4f` | Done | revision chain | Passed |
| CCH-T-01 | task brief | ≥2 variations per API/functionality | api + unit suites | Done | suite | Passed |
| CCH-SOR-26 | TASK-SOR-026 | Production buffer discipline | empty catalog `[]` + Redis port + buffer_class NONE | Implemented | `test_durable_sor` + `test_cch_redis_l2` | PASS |

**Coverage note:** HTTP namespace/TTL persist on `AsyncSession`; empty list is `[]`. `CacheCatalogStore` + MEMORY L2 are the TestClient double. Redis test-connection is `PROVIDER_PENDING` (no invented hits). `buffer_class=NONE` refuses set. Warmup/quotas/packs still memory.
