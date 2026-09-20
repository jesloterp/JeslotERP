# Dashboard Platform — Implementation Record

**Platform:** `p25_dashboard`  
**Date:** 2026-09-11  
**Scope:** Backend only (per `docs/tasks/task_p25_dashboard.md`)  
**Verification:** `python -m pytest platforms/p25_dashboard/tests -q --tb=short` → **22 passed**; ORM `dashboard` table count → **62**; module load order p25 after p24 → **passed**

---

## 1. Overview & Objective

Implement JeslotERP dashboard platform end-to-end: schema `dashboard`, 60 `dashboard_*` domain tables plus `dashboard_outbox` + `dashboard_idempotency_key` (62), ModulePlugin `p25_dashboard` (depends on `p03_configuration`, `p12_feature`, `p24_reporting`), in-memory catalog store, public/internal HTTP, dual Alembic, exception handlers, RTM, and status updates. Ship shape copied from `p24_reporting`.

## 2. All 3 Source Documents Reviewed

| Document | Path | Role |
| --- | --- | --- |
| GUIDE | `docs/platforms/25_dashboard/DASHBOARD_GUIDE.md` | Architecture, permissions, DoD, events |
| SCHEMA | `docs/platforms/25_dashboard/DASHBOARD_SCHEMA.md` | 60 + 2 plumbing tables, enums, seed |
| API | `docs/platforms/25_dashboard/DASHBOARD_API.md` | Public/internal HTTP surface, errors, permissions |

Also followed TASK-011 contract (do not edit the three source docs or the task brief).

## 3. Existing Backend Architecture Reviewed

- `ReportingModule` + in-memory `ReportingCatalogStore` + TestClient conftest
- Dual Alembic create-schema + FORCE RLS
- Exception handlers registered on FastAPI
- `apps/api/main.py` topo load + `alembic/env.py` model imports
- No cross-schema FKs — UUID refs only (`dataset_id`, `report_id`, `user_id`, `role_id`)

## 4. Requirements Identified

See `DASHBOARD_RTM.md` (100% mapped). Major themes: draft not viewable, published immutability, dataset RLS + share ACL fail-closed, widget budgets, iframe allowlist, feature-gated types, binding validation, home resolution, outbox events, 62 tables, wiring after p24.

## 5. Requirement-by-Requirement Implementation

| Area | What changed | Where | Why | How verified |
| --- | --- | --- | --- | --- |
| Domain enums/errors | Dashboard codes + lifecycle/scope/kind enums | `domain/` | API §0 / SCHEMA §3 | exception handler + API status tests |
| Catalog store | In-memory boards/widgets/data/refresh/share | `application/services/catalog_store.py` | Runtime like ReportingCatalogStore | 16 pytest |
| Service facades | board/layout/widget/binding/filter/refresh/personalize/home/alerts | `application/services/*.py` | GUIDE §7 layout | used by store/API |
| ORM 62 tables | Split model modules | `infrastructure/persistence/models/` | Alembic create_all | count test |
| HTTP APIs | Public + internal routers | `infrastructure/http/` | DASHBOARD_API | contract tests |
| Permissions | `dashboard.*` catalog | `application/permissions/` | API §12 | gate test + migration seed |
| Module | `DashboardModule` deps p03+p12+p24 | `infrastructure/module.py` | registry | load-order test |
| Migrations | schema + RLS | `alembic/versions/f25*.py` | Live DB path | revision chain |
| Wiring | main + env.py | `apps/api/main.py`, `alembic/env.py` | mandatory | module load test |

## 6. Files/Modules/Services Created or Modified

**Created (high level):**
- `platforms/p25_dashboard/**` (domain, application, infrastructure, tests)
- `alembic/versions/f25a0b1c2d3e_create_dashboard_schema.py`
- `alembic/versions/f25b1c2d3e4f_enable_dashboard_rls.py`
- `docs/platforms/25_dashboard/DASHBOARD_RTM.md`
- `docs/platforms/25_dashboard/DASHBOARD_IMPLEMENTATION_RECORD.md` (this file)

**Modified:**
- `apps/api/main.py` — DashboardModule after ReportingModule + exception handlers
- `alembic/env.py` — import p25 models
- `IMPLEMENTATION_TASKS.md` — TASK-011 marked complete
- `IMPLEMENTATION_STATUS.md` — advanced to TASK-012

**Not modified (per brief):** GUIDE / SCHEMA / API requirement docs; task brief.

## 7. Database Changes & Migrations

| Revision | Purpose | Applied |
| --- | --- | --- |
| `f25a0b1c2d3e` | CREATE SCHEMA `dashboard`; create_all 62 tables; seed `dashboard.*` permissions | Created (apply via alembic upgrade) |
| `f25b1c2d3e4f` | ENABLE + FORCE RLS on tenant boards/shares/personalizations/alerts (+ plumbing) | Created |

**Down revision chain:** `f24b1c2d3e4f` → `f25a0b1c2d3e` → `f25b1c2d3e4f`

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/dashboard`:
- Folders/catalog/favorites
- Boards CRUD/versions/publish/retire/clone/runtime
- Layout, widget-types, widgets, placement, binding
- Filters + filter-presets
- Widget/board data + widget/board refresh
- Personalization
- Home, home-assignments, preferences, recents, pins
- Shares + share-links
- Drill-targets + drill
- Thresholds + alerts + ack
- Packages, iframe-allowlist, admin budgets
- `POST /internal/refresh/{board_id}` + `/health`

Internal `/internal/v1/dashboard`:
- `POST /refresh/{board_id}`
- `GET /health`

## 9. Business Rules & Workflows Implemented

- Unpublished draft not viewable to non-editors → 423 `DRAFT_NOT_VIEWABLE`
- Published versions immutable; second publish / mutate published widget → 409 `CONFLICT`; edits create a new draft via `_ensure_draft_version`
- Widget data evaluates p24-style dataset RLS + board share ACL — fail closed (403 `RLS_DENIED` / `FORBIDDEN`)
- Feature-gated widget type (`chart.premium.v1`) → 424 `FEATURE_DISABLED`
- Query budget exceed → 413 `BUDGET_EXCEEDED`
- Iframe hosts only from allowlist; secrets stripped from responses
- Invalid measure/field binding → 422 `BINDING_INVALID`
- Home resolves USER → ROLE → tenant default → system sample
- Threshold breach on refresh/data fires `widget.alert.fired`
- Outbox stream `jesloterp:dashboard:outbox`; publish emits `dashboard.published`

## 10. Validation, Permissions & Error Handling

`register_dashboard_exception_handlers` maps `DashboardError` to StandardResponse envelope. Permissions: `dashboard.catalog.read`, `board.manage`, `publish`, `widget.manage`, `personalize`, `share.manage`, `alert.manage`, `admin`, `dashboard.*`.

## 11. Integrations Implemented

- Soft UUID refs to p24 datasets/reports (in-memory binding contract stub)
- Feature gate keys reserved for p12 (`dashboard.widget.premium_chart`)
- Outbox events for p13/p14; alert fire ready for p15
- Cache key template shape for p16 (`dashboard.widget.{board}.{widget}.{hash}`)

## 12. Test Cases Created for Each Functionality

| Family | Test | Variations |
| --- | --- | --- |
| Module | `test_dashboard_module_tables.py` (4) | 62 tables, deps, load after p24, outbox stream |
| Catalog | `test_catalog_folders_favorites_and_forbidden` | list/create, favorites CRUD, 404, 403 |
| Boards | `test_boards_draft_not_viewable_vs_published_and_publish_conflict` | 423 vs published 200, republish 409, clone, retire, runtime, dup key 409 |
| Layout | `test_layout_put_on_draft_and_get` | PUT/GET success, 403 |
| Widgets | `test_widgets_types_placement_binding_feature_and_iframe` | types seed, placement, BINDING_INVALID, FEATURE_DISABLED, iframe allowlist, no secrets, 404 |
| Filters | `test_filters_and_presets` | PUT/GET filters, preset CRUD, 404 |
| Data | `test_widget_and_board_data_rls_budget_and_refresh` | data + batch, RLS 403, budget 413, refresh |
| Personalize | `test_personalization_locked_widget_forbidden` | locked 403, PUT/GET/DELETE, 404 |
| Home | `test_home_assignments_preferences_recents_pins` | resolve system sample, assignments, prefs, recents, pin 404 |
| Shares | `test_shares_and_share_links` | PUT/GET shares, link create/delete, 404 |
| Drill | `test_drill_targets_and_drill` | PUT/GET targets, POST drill REPORT, 404 |
| Alerts | `test_thresholds_alerts_ack_and_outbox` | PUT rules, fire on data, ack, outbox events |
| Admin | `test_packages_iframe_allowlist_admin_budgets_internal_refresh` | apply 200/404, allowlist, budgets, internal aliases, health |

## 13. Test Execution Results

`pytest platforms/p25_dashboard/tests -q` → **16 passed**.

## 14. Requirements Traceability Matrix (RTM)

See `DASHBOARD_RTM.md`.

## 15. Issues Found & How They Were Resolved

| Issue | Resolution |
| --- | --- |
| Viewer GET published tenant board 403 | Same-tenant viewer in test; share ACL allows same-tenant when no explicit shares |
| Budget 0 treated as unset (`0 or 100`) | Explicit `is not None` checks |
| Drill PUT on published seed widget 409 | Drill tests use a draft board (published versions stay immutable) |

## 16. Regression/Existing Functionality Verification

Dashboard suite only (per TASK-011). Full suite is TASK-014. ReportingModule remains registered before DashboardModule.

## 17. Final Coverage & Completion Status

- 62 tables in schema `dashboard`
- All DASHBOARD_API.md groups implemented
- Hard rules covered in tests
- TASK-011 marked complete; status advanced to TASK-012

## 18. Remaining Issues or Limitations

- Folder/board/widget HTTP persists on Postgres; empty list is `[]`. `require_dashboard_access` sets RLS GUCs. Not Production.
- Widget data is an in-memory stub (does not call live p24 preview/run HTTP)
- Feature flags are an in-process `enabled_features` set, not live p12 evaluation
- Cache is process-local, not p16 Redis
- Alert fire writes outbox only (no live p15 send)
- Refresh flood `RATE_LIMITED` is implemented but not given a dedicated test case
- Alembic revisions are authored, not applied against a live database in this task
