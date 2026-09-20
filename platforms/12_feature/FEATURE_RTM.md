# Feature Platform — Requirements Traceability Matrix (RTM)

**Platform:** `p12_feature` · **Schema:** `feature`  
**Sources:** FEATURE_GUIDE.md · FEATURE_SCHEMA.md · FEATURE_API.md · docs/sample.md  
**Verification date:** 2026-09-12  
**Test command:** `python -m pytest platforms/p12_feature/tests -q --tb=short` → **57 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| FEAT-G-01 | GUIDE §1 | Feature flag control plane (not .env boolean) | `platforms/p12_feature` ModulePlugin | Done | `test_feat_module_*` | Passed |
| FEAT-G-02 | GUIDE §2 | Evaluate with env/rules/rollout/prereq | `FeatureCatalogStore.evaluate` | Done | `test_feat_evaluate_*`, precedence tests | Passed |
| FEAT-G-03 | GUIDE §3 | Flag-key stability, typed flags | Catalog CRUD + enums | Done | `test_feat_catalog_*`, multivariate | Passed |
| FEAT-G-04 | GUIDE §3 | Env-scoped delivery | env configs | Done | targeting/env API tests | Passed |
| FEAT-G-05 | GUIDE §3 | FIRST-match targeting | `_evaluate_one` rules loop | Done | `test_feat_segment_*`, clause tests | Passed |
| FEAT-G-06 | GUIDE §3 | Segments first-class | segments store + APIs | Done | `test_feat_segment_*` | Passed |
| FEAT-G-07 | GUIDE §3 | Sticky bucketing | `bucketing.sticky_bucket` | Done | `test_feat_sticky_*` | Passed |
| FEAT-G-08 | GUIDE §3 | Prerequisites + cycle | `prerequisite.py` + put_prerequisites | Done | `test_feat_prereq_*` | Passed |
| FEAT-G-09 | GUIDE §3/4 | Kill switch wins | `engage_kill` + evaluate | Done | `test_feat_kill_*`, precedence | Passed |
| FEAT-G-10 | GUIDE §4 | Overrides FORCE_* | overrides APIs | Done | `test_feat_tenant_override_*` | Passed |
| FEAT-G-11 | GUIDE §4 | Precedence order | evaluate pipeline | Done | `test_feat_precedence_*` | Passed |
| FEAT-G-12 | GUIDE §5 | Experiments + exposure | experiments + evaluate hook | Done | `test_feat_experiment_*` | Passed |
| FEAT-G-13 | GUIDE §7 | Permissions feature.* | permissions catalog + gates | Done | `test_feat_permission_*` | Passed |
| FEAT-G-14 | GUIDE §7 | RLS FORCE tenant tables | Alembic `f12b1c2d3e4f` | Done | migration applied | Passed (DB) |
| FEAT-G-15 | GUIDE §9 | Outbox `jesloterp:feature:outbox` | `FeatOutboxEvent` + store emit | Done | store emits on kill/override | Passed (unit via flows) |
| FEAT-S-01 | SCHEMA §2 | 60 domain + 3 plumbing = 63 tables | ORM models | Done | `test_feat_module_tables_count_63` | Passed |
| FEAT-S-02 | SCHEMA §1 | Schema name `feature` never p12 | `FEATURE_SCHEMA` | Done | module/tables tests | Passed |
| FEAT-S-03 | SCHEMA §1 | No cross-schema FKs | UUID refs only in models | Done | ORM review | Passed |
| FEAT-S-04 | SCHEMA §14 | Seed envs/flags/segments/permissions | store seed + Alembic seed | Done | defaults in API tests | Passed |
| FEAT-A-01 | API §4 | Error codes FEAT_* | `domain/exceptions.py` + handlers | Done | 404/409/422 API tests | Passed |
| FEAT-A-02 | API §5 | Permission codes | `FEATURE_PERMISSIONS` + migration seed | Done | permission gate tests | Passed |
| FEAT-A-03 | API §6.1 | POST /evaluate | evaluate router | Done | `test_feat_evaluate_*` | Passed |
| FEAT-A-04 | API §6.2 | GET flag evaluate sugar | evaluate router | Done | covered via evaluate | Passed |
| FEAT-A-05 | API §6.3 | Bootstrap ETag 304 | bootstrap | Done | `test_feat_bootstrap_*` | Passed |
| FEAT-A-06 | API §6.4 | Internal evaluate | internal router | Done | `test_feat_internal_*` | Passed |
| FEAT-A-07 | API §7 | Flag catalog CRUD/archive/variations | catalog router | Done | `test_feat_catalog_*` | Passed |
| FEAT-A-08 | API §8 | Environments & targeting | targeting router | Done | clause/env tests | Passed |
| FEAT-A-09 | API §9 | Segments + members batch | segments router | Done | `test_feat_segment_*` | Passed |
| FEAT-A-10 | API §10 | Kill + clear + break-glass | kill_overrides router | Done | kill/break-glass tests | Passed |
| FEAT-A-11 | API §11 | Tenant/company overrides | kill_overrides router + `persist_override` | Done | override tests + `test_feat_overrides_list_db_first` | Passed |
| FEAT-SOR-01 | TASK-SOR-014 | Flag/override HTTP → Postgres; empty list `[]` | `flag_repository` + `require_feature_access` | Done | `test_durable_sor` empty/persist-then-fetch | Passed |
| FEAT-A-12 | API §12 | Schedules + tick + promote | governance + internal | Done | schedule/promote tests | Passed |
| FEAT-A-13 | API §13 | Experiments start/stop/summary | governance | Done | experiment tests | Passed |
| FEAT-A-14 | API §14 | Packages + changesets/approvals | governance | Done | package/promote approval | Passed |
| FEAT-A-15 | API §15 | Audit/stats/eval-samples | governance | Done | `test_feat_audit_and_stats` | Passed |
| FEAT-A-16 | API §16 | Webhooks + SDK keys (secret_ref) | governance | Done | `test_feat_sdk_key_*` | Passed |
| FEAT-W-01 | Wiring | main.py FeatureModule + handlers | `apps/api/main.py` | Done | `test_feat_app_loads_p12_after_p11` | Passed |
| FEAT-W-02 | Wiring | alembic env import models | `alembic/env.py` | Done | upgrade applied | Passed |
| FEAT-W-03 | Wiring | Migrations revise rules head | `f12a0b1c2d3e` → `f12b1c2d3e4f` | Done | `alembic current` | Passed |
| FEAT-T-01 | sample.md | ≥2 variations per area | unit/api/engine/bucketing/targeting/permissions | Done | 57 tests | Passed |

**Coverage note:** Flag/kill/override HTTP is Postgres-first (`[]` on empty `AsyncSession`). Evaluate/bootstrap stay on in-memory `FeatureCatalogStore` (TestClient/`AsyncMock` double). Segments, schedules, experiments, break-glass, and p26 license compile remain follow-on. Not Production.
