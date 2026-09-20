# Dashboard Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p25_dashboard/tests -q --tb=short` → **22 passed**; 62 `dashboard` tables; load order after p24.

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| DSH-G-01 | GUIDE §1 | Interactive board & widget control plane | `DashboardCatalogStore` + schema `dashboard` | Implemented | module tables + health | PASS |
| DSH-G-02 | GUIDE §1 | Catalog, layouts, widgets, bindings, filters, refresh, personalization, share, home, drill, alerts, packs | store + public APIs | Implemented | all API families | PASS |
| DSH-G-03 | GUIDE §1 | Does not own datasets/exports (p24), SLO probes (p21), feature eval (p12), settings (p03) | UUID refs / stubs only | Implemented | binding + feature gate | PASS |
| DSH-G-04 | GUIDE §2 | No client SQL; bind p24 datasets/reports | binding validator | Implemented | BINDING_INVALID 422 | PASS |
| DSH-G-05 | GUIDE §2 | Widget data inherits dataset RLS + board share ACL — fail closed | `evaluate_dataset_rls` + share ACL | Implemented | RLS_DENIED 403 | PASS |
| DSH-G-06 | GUIDE §2 | Heavy queries budgeted; cache key includes tenant/user/filters | query budgets + widget cache | Implemented | BUDGET_EXCEEDED 413 | PASS |
| DSH-G-07 | GUIDE §2 | Personalization cannot escalate beyond share / locked widgets | `put_personalization` 403 | Implemented | locked widget 403 | PASS |
| DSH-G-08 | GUIDE §2 | No cross-schema FKs | UUID refs on ORM | Implemented | table inventory | PASS |
| DSH-G-09 | GUIDE §2 | System boards admin; tenants clone | clone + lineage | Implemented | clone + package apply | PASS |
| DSH-G-10 | GUIDE §2 | Publish versioned; published immutable; edits create draft | publish + `_assert_mutable_version` + `_ensure_draft_version` | Implemented | publish then 409 | PASS |
| DSH-G-11 | GUIDE §3 | Feature-gated widget types | `chart.premium.v1` + `enabled_features` | Implemented | FEATURE_DISABLED 424 | PASS |
| DSH-G-12 | GUIDE §3 | Outbox `dashboard.published`, `widget.alert.fired` | `_emit` + `jesloterp:dashboard:outbox` | Implemented | outbox + stream | PASS |
| DSH-G-13 | GUIDE §6 | Permissions `dashboard.*` | `require_dashboard_permission` | Implemented | catalog 403 | PASS |
| DSH-G-14 | GUIDE §6 | FORCE RLS tenant boards/shares/personalizations/alerts | Alembic `f25b1c2d3e4f` | Implemented | migration present | PASS |
| DSH-G-15 | GUIDE §6 | Running-user widget data; share VIEW ≠ dataset manage | widget data + RLS | Implemented | RLS deny | PASS |
| DSH-G-16 | GUIDE §10 | DoD: no SQL, RLS, personalization lock, published immutable, home resolve, cache keys, feature gates, no XFKs | store + APIs | Implemented | hard-rule tests | PASS |
| DSH-S-01 | SCHEMA §1 | Schema `dashboard` (never p25); `dashboard_*` tables | `DASHBOARD_SCHEMA` | Implemented | table names | PASS |
| DSH-S-02 | SCHEMA §2 | 60 domain + outbox + idempotency = 62 | ORM models | Implemented | `test_dashboard_module_tables_count_62` | PASS |
| DSH-S-03 | SCHEMA §3 | Enums lifecycle/scope/kind/chart/refresh/share/drill/threshold | `domain/enums.py` | Implemented | API payloads | PASS |
| DSH-S-04 | SCHEMA §4 | Board + version + locale + default filter + theme + lifecycle + lineage | board models/store | Implemented | boards family | PASS |
| DSH-S-05 | SCHEMA §5 | Widget type/schema/instance/config/placement/title/visibility/action/feature/iframe | widget models/store | Implemented | widgets family | PASS |
| DSH-S-06 | SCHEMA §6 | Binding + fields/measures + filters + sync + query budget | binding models/store | Implemented | filters + binding | PASS |
| DSH-S-07 | SCHEMA §7 | Refresh policy/widget refresh/cache template/job/stale | refresh models/store | Implemented | refresh APIs | PASS |
| DSH-S-08 | SCHEMA §8 | Personalization/home/prefs/recents/pins | personalize models/store | Implemented | home/prefs family | PASS |
| DSH-S-09 | SCHEMA §9 | Share/drill/threshold/alert/link | security models/store | Implemented | shares/drill/alerts | PASS |
| DSH-S-10 | SCHEMA §10 | Packs ops/finance/fleet; clone lineage | seed packages | Implemented | packages apply | PASS |
| DSH-S-11 | SCHEMA §11 | `dashboard_outbox`, `dashboard_idempotency_key` | plumbing models | Implemented | table names + stream | PASS |
| DSH-S-12 | SCHEMA §12 | FORCE tenant isolation; SYSTEM published readable | Alembic FORCE + NULL tenant seed | Implemented | seed catalog + viewer publish | PASS |
| DSH-S-13 | SCHEMA §13 | Seed widget types, refresh policies, sample board, perms, budgets, `core.samples.v1` | `seed_defaults` + Alembic perms | Implemented | types + packages + published board | PASS |
| DSH-S-14 | SCHEMA §15 | Split models catalog/board/layout/widget/binding/refresh/personalize/security/governance/plumbing | persistence package | Implemented | table count | PASS |
| DSH-A-00 | API §0 | Envelope + FORBIDDEN/RLS_DENIED/NOT_FOUND/CONFLICT/VALIDATION_ERROR/BINDING_INVALID/DRAFT_NOT_VIEWABLE/FEATURE_DISABLED/BUDGET_EXCEEDED/RATE_LIMITED | `domain/exceptions.py` | Implemented | all API families | PASS |
| DSH-A-01 | API §1 | Folders + catalog + favorites | `/folders` `/catalog` `/favorites` | Implemented | catalog family | PASS |
| DSH-A-02 | API §2 | Boards CRUD/versions/publish/retire/clone/runtime | `/boards*` | Implemented | boards family | PASS |
| DSH-A-03 | API §3 | Layout + widget types/instances/placement/binding | `/layout` `/widgets*` `/widget-types` | Implemented | layout + widgets | PASS |
| DSH-A-04 | API §4 | Filters + presets | `/filters` `/filter-presets` | Implemented | filters family | PASS |
| DSH-A-05 | API §5 | Widget/board data + refresh + internal refresh | `/data` `/refresh` `/internal/refresh` | Implemented | data + refresh | PASS |
| DSH-A-06 | API §6 | Personalization GET/PUT/DELETE | `/personalization` | Implemented | personalization family | PASS |
| DSH-A-07 | API §7 | Home / assignments / prefs / recents / pins | `/home*` `/preferences` `/recents` `/pins` | Implemented | home family | PASS |
| DSH-A-08 | API §8 | Shares + share-links | `/shares` `/share-links` | Implemented | shares family | PASS |
| DSH-A-09 | API §9 | Drill-targets + drill | `/drill-targets` `/drill` | Implemented | drill family | PASS |
| DSH-A-10 | API §10 | Thresholds + alerts + ack | `/thresholds` `/alerts` | Implemented | alerts family | PASS |
| DSH-A-11 | API §11 | Packages + iframe allowlist + admin budgets | `/packages` `/iframe-allowlist` `/admin/budgets` | Implemented | admin family | PASS |
| DSH-A-12 | API §12 | Permission matrix | `dashboard.*` codes | Implemented | 403 + admin gates | PASS |
| DSH-W-01 | TASK-011 | ModulePlugin deps p03/p12/p24; load after p24 | `DashboardModule` + `main.py` | Implemented | module tests | PASS |
| DSH-W-02 | TASK-011 | Dual Alembic f25a revises f24b; f25b FORCE RLS | `alembic/versions/f25*` | Implemented | revision chain | PASS |
| DSH-W-03 | TASK-011 | Public `/api/v1/dashboard` + internal aliases + health | `api_v1.py` | Implemented | health + internal refresh | PASS |
| DSH-SOR-01 | TASK-SOR-022 | Folder/board/widget HTTP → Postgres; empty `[]` | `catalog_repository` + `require_dashboard_access` | Implemented | persist-then-fetch + db-first empty | PASS |
