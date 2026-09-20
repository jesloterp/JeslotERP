# Reporting Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p24_reporting/tests -q --tb=short` → **24 passed**; 64 `reporting` tables; load order after p23.

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| RPT-G-01 | GUIDE §1 | Report definition, dataset, export control plane | `ReportingCatalogStore` + schema `reporting` | Implemented | module tables + health | PASS |
| RPT-G-02 | GUIDE §1 | Governed catalog, datasets, defs, params, execution, exports, subscriptions, RLS, snapshots, discoverability | store + public APIs | Implemented | all API families | PASS |
| RPT-G-03 | GUIDE §1 | Does not own boards (p25), search SoR (p18), media bytes (p08), cron (p17) | stubs/refs only | Implemented | download signed URL stub; reindex target p18 | PASS |
| RPT-G-04 | GUIDE §2 | Reports never bypass dataset RLS — fail closed | `evaluate_rls` + `RlsDeniedError` | Implemented | preview RLS 403 | PASS |
| RPT-G-05 | GUIDE §2 | Export bytes in p08; store media_id + checksum | artifact + signed URL stub | Implemented | export download | PASS |
| RPT-G-06 | GUIDE §2 | Heavy runs async (p14 job_id); sync for small | `mode` SYNC/ASYNC/AUTO | Implemented | sync SUCCEEDED vs async QUEUED | PASS |
| RPT-G-07 | GUIDE §2 | No ad-hoc SQL from clients | sandboxed compile | Implemented | DROP TABLE 422 | PASS |
| RPT-G-08 | GUIDE §2 | No cross-schema FKs | UUID refs on ORM | Implemented | table inventory | PASS |
| RPT-G-09 | GUIDE §2 | Definition versioned; published vs draft | dataset/report versions | Implemented | draft 423 vs published run | PASS |
| RPT-G-10 | GUIDE §2 | Period-close snapshots immutable | `seal_snapshot` 409 | Implemented | seal conflict | PASS |
| RPT-G-11 | GUIDE §3 | Dataset-first; compile then run; query budgets; outbox events | compilers + budget + `_emit` | Implemented | compile / budget / outbox | PASS |
| RPT-G-12 | GUIDE §6 | Permissions `reporting.*` | `require_reporting_permission` | Implemented | catalog 403 | PASS |
| RPT-G-13 | GUIDE §6 | FORCE RLS tenant defs/runs/exports/shares/snapshots | Alembic `f24b1c2d3e4f` | Implemented | migration present | PASS |
| RPT-G-14 | GUIDE §6 | Secrets none; downloads via signed URL | download grants | Implemented | signed_url stub | PASS |
| RPT-G-15 | GUIDE §8 | Events dataset/def/run/export/subscription/snapshot/share | outbox `_emit` | Implemented | store outbox + stream | PASS |
| RPT-G-16 | GUIDE §10 | DoD: no SQL, RLS fail-closed, media_id, schedule idempotency, draft not runnable, sealed immutable, budgets, no XFKs | store + APIs | Implemented | hard-rule tests | PASS |
| RPT-S-01 | SCHEMA §1 | Schema `reporting` (never p24); `reporting_*` tables | `REPORTING_SCHEMA` | Implemented | table names | PASS |
| RPT-S-02 | SCHEMA §2 | 62 domain + outbox + idempotency = 64 | ORM models | Implemented | `test_reporting_module_tables_count_64` | PASS |
| RPT-S-03 | SCHEMA §3 | Enums lifecycle/layout/field/agg/run/export/share/param | `domain/enums.py` | Implemented | API payloads | PASS |
| RPT-S-04 | SCHEMA §4 | Dataset + version + field + measure + RLS + budget | dataset models/store | Implemented | datasets family | PASS |
| RPT-S-05 | SCHEMA §5 | Report + version + columns | report models/store | Implemented | reports family | PASS |
| RPT-S-06 | SCHEMA §6 | Parameters + variants | param models/store | Implemented | variants family | PASS |
| RPT-S-07 | SCHEMA §7 | Runs + exports + artifacts (media_id) | execution/export models | Implemented | runs + exports | PASS |
| RPT-S-08 | SCHEMA §8 | Subscriptions + schedule_binding + burst + fire history | subscription models | Implemented | subscriptions family | PASS |
| RPT-S-09 | SCHEMA §9 | Shares + snapshots sealed immutable | security models | Implemented | shares + snapshots | PASS |
| RPT-S-10 | SCHEMA §10 | Packs freight/gst/fleet; feature bindings PARQUET/PDF | seed packages | Implemented | packages apply | PASS |
| RPT-S-11 | SCHEMA §11 | `reporting_outbox`, `reporting_idempotency_key` | plumbing models | Implemented | table names + stream | PASS |
| RPT-S-12 | SCHEMA §12 | FORCE tenant isolation; system packs readable | Alembic FORCE + NULL tenant seed | Implemented | seed catalog list | PASS |
| RPT-S-13 | SCHEMA §13 | Seed folders Company/Ops/Finance/Fleet; sample TABLE + variant; budgets 5k/10s + 1M/5min; core.samples.v1; perms; CSV/XLSX/PDF | `seed_defaults` + Alembic perms | Implemented | folders + report-types + packages | PASS |
| RPT-S-14 | SCHEMA §15 | Split models catalog/dataset/report/param/execution/export/subscription/security/governance/plumbing | persistence package | Implemented | table count | PASS |
| RPT-A-00 | API §0 | Envelope + errors INVALID_PARAMETERS/FORBIDDEN/RLS_DENIED/NOT_FOUND/CONFLICT/BUDGET_EXCEEDED/VALIDATION_ERROR/DRAFT_NOT_RUNNABLE/RATE_LIMITED/RUN_CANCELLED | `domain/exceptions.py` | Implemented | all API families | PASS |
| RPT-A-01 | API §1 | Folders CRUD + catalog browse + favorites | `/folders` `/catalog` `/favorites` | Implemented | catalog family | PASS |
| RPT-A-02 | API §2 | Datasets CRUD/versions/publish/fields/rls/budget/preview | `/datasets*` | Implemented | datasets family | PASS |
| RPT-A-03 | API §3 | Reports CRUD/versions/publish/retire/compile/move/tags | `/reports*` | Implemented | reports family | PASS |
| RPT-A-04 | API §4 | Parameters + variants | `/reports/{id}/parameters` `/variants` | Implemented | variants family | PASS |
| RPT-A-05 | API §5 | Runs start/list/get/pages/metrics/cancel; internal complete | `/runs*` `/internal/runs/{id}/complete` | Implemented | runs + cancel + complete | PASS |
| RPT-A-06 | API §6 | Exports create/status/download/deliver | `/exports*` | Implemented | exports family | PASS |
| RPT-A-07 | API §7 | Subscriptions CRUD/pause/resume/history + internal fire | `/subscriptions*` `/internal/subscriptions/{id}/fire` | Implemented | subscriptions family | PASS |
| RPT-A-08 | API §8 | Shares PUT + share-links | `/shares` `/share-links` | Implemented | shares family | PASS |
| RPT-A-09 | API §9 | Snapshots create/list/get/seal/download; seal 409 | `/snapshots*` | Implemented | snapshot seal conflict | PASS |
| RPT-A-10 | API §10 | Packages apply; admin budgets; purge-runs; report-types | `/packages*` `/admin/*` `/report-types` | Implemented | admin family | PASS |
| RPT-A-11 | API §11 | Dataset binding-contract; internal catalog reindex | `/binding-contract` `/internal/catalog/reindex` | Implemented | binding + reindex aliases | PASS |
| RPT-A-12 | API §12 | Permission matrix | HTTP gates | Implemented | 403 catalog | PASS |
| RPT-MOD | registry / brief | ModulePlugin after p23; deps p02+p05+p18; Alembic f24a/f24b; outbox `jesloterp:reporting:outbox`; `/internal/v1/reporting/*` + health | `ReportingModule` + main/env | Implemented | load order + deps + stream | PASS |
| RPT-SOR-01 | TASK-SOR-022 | Dataset/report HTTP → Postgres; empty `[]` | `catalog_repository` + `require_reporting_access` | Implemented | persist-then-fetch + db-first empty | PASS |
| RPT-SOR-02 | TASK-SOR-022 | DWH port; no fake warehouse | `DwhWarehouseEngine` | Implemented | PROVIDER_PENDING ping | PASS |
