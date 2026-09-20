# Scheduler Platform — Implementation Record

**Platform:** `p17_scheduler`  
**Date:** 2026-09-11  
**Scope:** Backend only (per `docs/tasks/task_p17_scheduler.md`)  
**Verification:** `pytest platforms/p17_scheduler/tests -q --tb=line` → **27 passed**; ORM `scheduler` table count → **60**; module load order p17 after p16 → **passed**

---

## 1. Overview & Objective

Implement JeslotERP scheduler control plane end-to-end: schema `scheduler`, 60 `sch_*` tables (58 domain + outbox + idempotency), ModulePlugin `p17_scheduler` (depends on `p01_identity`, `p14_messaging`), cron/interval/one-shot/window/dependent schedules, timezone-correct next-fire, misfire/overlap/blackout/catch-up, cluster tick locks, enqueue-only p14 bridge, run ledger + callbacks, packs, permissions, RLS migrations, tests, RTM.

## 2. All 3 Source Documents Reviewed

| Document | Path | Role |
| --- | --- | --- |
| GUIDE | `docs/platforms/17_scheduler/SCHEDULER_GUIDE.md` | Architecture, DoD, anti-patterns |
| SCHEMA | `docs/platforms/17_scheduler/SCHEDULER_SCHEMA.md` | 58 + 2 plumbing tables, enums, seed |
| API | `docs/platforms/17_scheduler/SCHEDULER_API.md` | Public/internal HTTP, errors, permissions |

Also followed `docs/tasks/task_p17_scheduler.md` and mirrored `platforms/p16_cache/`.

## 3. Existing Backend Architecture Reviewed

- ModulePlugin registration and topo-sort deps in `apps/api/main.py`
- Alembic `env.py` dynamic model imports
- p16 patterns: in-memory catalog store, thin routers, exception handlers, permission deps, outbox stream, dual migrations (schema + RLS)
- Shared `EnterpriseBase` / `PlatformBase` / catalog base ORM bases
- No cross-schema FKs — UUID refs only
- p14 treated as available (enqueue recorded as job stub; no handler execution in ticker)

## 4. Requirements Identified

See `SCHEDULER_RTM.md` (100% mapped).

## 5. Requirement-by-Requirement Implementation

| Area | What changed | Where | Why | How verified |
| --- | --- | --- | --- | --- |
| Domain enums/errors | SCH_* codes + kind/run/misfire/overlap | `domain/` | API §4 / SCHEMA §3 | exception handler + API status tests |
| Cron / next-fire | 5/6-field cron, IST/DST | `cron_parser.py`, `next_fire.py` | GUIDE DoD | cron unit tests |
| Catalog store | In-memory control plane + tick | `catalog_store.py` | Runtime like CacheCatalogStore | 27 pytest |
| Services | misfire, overlap, blackout, lock, catch_up, enqueue_bridge, tick_engine | `application/services/` | GUIDE §8 layout | unit tests |
| ORM 60 tables | Split model modules | `infrastructure/persistence/models/` | SCHEMA §2 | count test |
| HTTP APIs | Public + internal routers | `infrastructure/http/` | SCHEDULER_API | contract tests |
| Permissions | `scheduler.*` catalog | `application/permissions/` | API §5 | 403 gate test + migration seed |
| Module | `SchedulerModule` deps p01+p14 | `infrastructure/module.py` | registry | load-order test |
| Migrations | schema + RLS | `alembic/versions/f17*.py` | Live DB path | revision chain |
| Wiring | main + env.py | `apps/api/main.py`, `alembic/env.py` | mandatory | module load test |

## 6. Files/Modules/Services Created or Modified

**Created:** `platforms/p17_scheduler/**` (domain, services, models, HTTP, workers, tests)  
**Created:** `alembic/versions/f17a0b1c2d3e_create_scheduler_schema.py`, `f17b1c2d3e4f_enable_scheduler_rls.py`  
**Created:** `docs/platforms/17_scheduler/SCHEDULER_RTM.md`, this record  
**Modified:** `apps/api/main.py` (SchedulerModule + exception handlers), `alembic/env.py` (model import)

**Not modified (per brief):** GUIDE / SCHEMA / API requirement docs; task brief.

## 7. Database Changes & Migrations

- Schema `scheduler` created via `f17a0b1c2d3e` (`create_all` on 60 tables)
- Seeds: UTC + Asia/Kolkata, misfire/overlap policies, handler allowlist, `scheduler.*` permissions
- RLS FORCE on tenant-scoped tables via `f17b1c2d3e4f` (schedule, run, manual_trigger, blackout, calendar, quota, usage, idempotency)

## 8. APIs/Endpoints Implemented

Public `/api/v1/scheduler`: schedules CRUD/versions/activate/pause/resume/retire, simulate, run-now, runs, calendars/blackouts, dependencies, policies, handlers, quotas, packages, changesets, stats, tickers, health, audit.

Internal `/internal/v1/scheduler`: tick, tick/dry-run, locks/heartbeat, shards/rebalance, run job-started/succeeded/failed.

## 9. Business Rules & Workflows

Tick: lock shard → due scan by `next_fire_at` → blackout/calendar/feature/dependency/overlap/misfire → create planned+run → enqueue p14 stub → advance next_fire. Manual run audited + idempotent. Pause stops claims; inflight continue. Unique planned fire (skips offset microseconds to keep ledger).

## 10. Validation, Permissions & Error Handling

`SchedulerError` → JSON envelope. Permissions `scheduler.*` with platform-admin bypass. Handler allowlist required before create/activate.

## 11. Integrations

Enqueue-only bridge to p14 (`handler_key`/`queue_key`/payload templates). Optional p13 event types written to outbox stream `jesloterp:scheduler:outbox`. No live Redis/DB ticker in unit tests.

## 12. Test Cases

Module (4), cron (4), misfire (2), overlap (2), lock (2), API contracts (13 groups, ≥2 cases each).

## 13. Test Execution Results

`pytest platforms/p17_scheduler/tests -q --tb=line` → **27 passed**.

## 14. RTM

See `SCHEDULER_RTM.md`.

## 15. Issues Found & Resolved

- Dataclass field `rebalance` shadowed method → renamed `rebalance_events`
- Serialized version ids are strings → `_get_version` accepts UUID or str
- Overlap skip vs unique planned fire → skip fires use microsecond offset

## 16. Regression

p17 load-order test instantiates full `load_modules()` including p16. p16 suite not re-run this task (unchanged). Full suite is TASK-014.

## 17. Final Coverage & Completion Status

TASK-003 acceptance bar met: schema, ModulePlugin, public+internal API, Alembic, ≥2 test variations per API group, RTM, implementation record.

## 18. Remaining Issues or Limitations

- Calendar/blackout HTTP and ticker/lock persist on `AsyncSession`. Tick hydrates calendars from Postgres. Schedule/admin catalog still uses the in-memory store as a TestClient double. Not Production.
- p14 enqueue is a stub job id (no worker callback unless tests/internal APIs invoke it)
- Cron walker is minute/second stepping (fine for tests; production may want a tighter calendar algorithm)
- Quota hard-block is on create count, not every tick fire
