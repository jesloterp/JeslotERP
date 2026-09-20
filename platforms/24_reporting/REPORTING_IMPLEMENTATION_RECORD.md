# Reporting Platform — Implementation Record

**Platform:** `p24_reporting`  
**Date:** 2026-09-12  
**Verification:** `python -m pytest platforms/p24_reporting/tests -q --tb=short` → **24 passed**; 64 `reporting` tables; load order after p23.

## 1. Overview & Objective

Report definition, dataset, and export control plane: schema `reporting`, governed catalog/folders, datasets + compiler + RLS, report defs + variants, in-memory execution stub, exports (media_id + signed URL stubs), subscriptions (p17 fire idempotency), shares, snapshots, packs/admin. Alembic f24a/f24b. In-memory catalog store matching p22/p23. Warehouse engine stubbed (no client SQL).

## 2. Source documents reviewed

`REPORTING_GUIDE.md`, `REPORTING_SCHEMA.md`, `REPORTING_API.md` under `docs/platforms/24_reporting/` plus `docs/tasks/task_p24_reporting.md`. GUIDE / SCHEMA / API updated to **SoR-Live** for TASK-SOR-022.

## 3. Existing backend architecture reviewed

p22_api / p23_integration ModulePlugin, in-memory catalog store, exception handlers from p05 auth, `require_internal_token`, dual Alembic schema+RLS, TestClient factory. Same layout copied for p24. Depends on p02_organization, p05_metadata, p18_search.

## 4. Requirements identified

See `REPORTING_RTM.md` (100% of GUIDE / SCHEMA / API items mapped).

## 5. Requirement-by-requirement implementation

In-memory `ReportingCatalogStore` implements folders/catalog/favorites, datasets (compile/publish/RLS/budget/preview/binding-contract), reports (versions/publish/retire/compile/move/tags), parameters/variants, runs (SYNC/ASYNC/AUTO + pages/metrics/cancel + internal complete), exports/download/deliver, subscriptions (pause/resume/history + idempotent fire), shares/share-links, snapshots/seal, packages/admin budgets/purge, catalog reindex. HTTP is thin CQRS over the store. Public `/api/v1/reporting` plus documented `/api/v1/reporting/internal/*` and `/internal/v1/reporting/*` aliases + health.

Hard rules: unpublished draft → 423 `DRAFT_NOT_RUNNABLE`; dataset RLS deny → 403 `RLS_DENIED`; budget exceed → 413 `BUDGET_EXCEEDED`; sealed snapshot mutate → 409 `CONFLICT`; invalid params → 400 `INVALID_PARAMETERS`; cancelled run pages/export/complete → 409 `RUN_CANCELLED`; compile fail → 422 `VALIDATION_ERROR`; raw SQL rejected.

## 6. Files created or modified

**Created**

- `platforms/p24_reporting/` domain, application (permissions + services), infrastructure HTTP/module/persistence, tests
- `platforms/p24_reporting/infrastructure/persistence/schema_constants.py` (`REPORTING_SCHEMA = "reporting"`)
- 62 `reporting_*` domain tables + `reporting_outbox` + `reporting_idempotency_key` = **64**
- `alembic/versions/f24a0b1c2d3e_create_reporting_schema.py`
- `alembic/versions/f24b1c2d3e4f_enable_reporting_rls.py`
- `docs/platforms/24_reporting/REPORTING_RTM.md`
- this record

**Modified**

- `apps/api/main.py` — `ReportingModule` after `IntegrationModule`; exception handlers
- `alembic/env.py` — import p24 models
- `IMPLEMENTATION_TASKS.md`, `IMPLEMENTATION_STATUS.md`

**Not edited**

- `docs/platforms/24_reporting/REPORTING_GUIDE.md`, `REPORTING_SCHEMA.md`, `REPORTING_API.md`
- `docs/tasks/task_p24_reporting.md`

## 7. Database changes & migrations

Schema `reporting`, 62 domain `reporting_*` tables + `reporting_outbox` + `reporting_idempotency_key` = **64**. FORCE RLS on tenant defs (dataset/report + versions), runs (+ children), exports (+ artifacts/deliveries/grants), shares, snapshots (+ row meta), outbox, idempotency. Permissions `reporting.*` seeded in f24a. No cross-schema FKs.

## 8. APIs

Public `/api/v1/reporting` (API §1–11). Internal complete/fire/reindex at `/api/v1/reporting/internal/*` and `/internal/v1/reporting/*`. Health at `/api/v1/reporting/health` and `/internal/v1/reporting/health`.

## 9. Business rules & workflows

- Unpublished draft reports cannot run in prod → 423 `DRAFT_NOT_RUNNABLE`
- Dataset RLS deny-by-default when rules exist and no allow matches (non-admin) → 403 `RLS_DENIED`
- Budget exceed (rows/time/bytes, including sync_max_rows) → 413 `BUDGET_EXCEEDED`
- Sealed snapshot cannot be mutated → 409 `CONFLICT`
- Invalid/missing required run params → 400 `INVALID_PARAMETERS`
- Cancelled run pages/export/complete → 409 `RUN_CANCELLED`
- Compile validates definition against dataset fields/measures → 422 `VALIDATION_ERROR`
- Expression DSL rejects raw SQL tokens
- Secrets none; downloads via signed URL stubs + `download_grant`
- Subscription fire idempotent per `idempotency_window_key`
- Seed: folders Company/Operations/Finance/Fleet; report type `table.standard.v1`; dataset `identity.users.count` published; report `sample.users.table` published + variant; budgets sync 5k/10s async 1M/5min; package `core.samples.v1`
- Outbox stream `jesloterp:reporting:outbox`
- Engine is an in-memory stub (not a warehouse)

## 10. Validation, permissions & errors

API §0 codes as `exception.code`. Permissions from GUIDE §6 / API §12 via `require_reporting_permission`.

## 11. Integrations

Depends on `p02_organization`, `p05_metadata`, `p18_search` (module graph). Domain events go to outbox `jesloterp:reporting:outbox`. p14 job ids allocated in-store for ASYNC (no live worker). p17 schedule ids allocated on subscription create; fire is the documented internal hook. p15 deliver recorded as notify stub. p08 media_id + signed URL stub. p18 reindex writes in-memory docs. p25 binding-contract returns stable field/measure/param/RLS notes.

## 12. Tests

Module (4) + API families (9) with ≥2 variations each (success + failure/guard). **13 passed**.

Covered: 64 tables; deps `[p02_organization, p05_metadata, p18_search]`; load after p23; outbox stream; draft not runnable vs published run; RLS deny; budget exceed; snapshot seal conflict; cancel; catalog list.

## 13. Test execution results

`pytest platforms/p24_reporting/tests -q` → **13 passed**.

## 14. RTM

`REPORTING_RTM.md`.

## 15. Issues found & how they were resolved

`sync_max_rows=0` / `max_rows=0` was treated as missing because `value or default` treats 0 as falsy; `_guard_budget` now uses an explicit None check so a zero cap returns 413 `BUDGET_EXCEEDED`. RLS deny tests use a non-`reporting.*` designer principal so admin bypass does not hide fail-closed policy.

## 16. Regression / existing functionality

p24 suite green. Full-repo suite not run (TASK-014).

## 17. Final coverage & completion status

TASK-SOR-022 acceptance bar met for p24: implement + tests + GUIDE/SCHEMA/API + RTM + this record. Not Production.

## 18. Remaining issues or limitations

1. Dataset/report HTTP persists on Postgres. Empty list is `[]`. `require_reporting_access` sets RLS GUCs. Folders/runs/exports/subscriptions still memory. Not Production.
2. Pytest warehouse is SQLITE. Production DWH is `PROVIDER_PENDING` (no invented hits).
