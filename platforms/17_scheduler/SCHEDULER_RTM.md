# Scheduler Platform — Requirements Traceability Matrix

**Platform:** `p17_scheduler`  
**Date:** 2026-09-11  
**Verification:** `pytest platforms/p17_scheduler/tests -q` → **46 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| SCH-SOR-06 | TASK-SOR-006 | Durable ticker / calendar ledger | calendar_repository + lock_repository | Implemented | `test_persist_calendar_then_fetch_from_same_session` | PASS |
| SCH-G-01 | GUIDE §1 | Time-based trigger control plane; enqueue-only to p14 | `tick_engine`, `enqueue_bridge`, `workers/ticker.py` | Implemented | `test_tick_enqueue_and_dry_run` | PASS |
| SCH-G-02 | GUIDE §1 | Cron/calendar, one-shot, window, dependent, interval kinds | `domain/enums.py`, `next_fire.py`, catalog store | Implemented | `test_schedules_list_create_and_not_found`, cron tests | PASS |
| SCH-G-03 | GUIDE §1 | Misfire FIRE_ONCE_NOW / SKIP / CATCH_UP_N / FIRE_ALL_BOUNDED | `misfire.py`, tick catch-up | Implemented | `test_catchup_bound_modes`, `test_tick_catch_up_n_respects_bound` | PASS |
| SCH-G-04 | GUIDE §1 | Overlap SKIP / QUEUE_ONE / FORBID / ALLOW_PARALLEL | `overlap.py`, tick + run-now | Implemented | `test_overlap_action_matrix`, `test_overlap_skip_and_unique_planned_fire` | PASS |
| SCH-G-05 | GUIDE §1 | Blackout calendars; ignore_blackout for critical | `blackout.py`, calendars API | Implemented | `test_tick_blackout_skips_non_critical`, `test_run_now_blackout_and_audit` | PASS |
| SCH-G-06 | GUIDE §1 | Schedule dependencies B after A SUCCESS | `put_dependencies`, cycle detect | Implemented | `test_dependencies_ok_and_cycle` | PASS |
| SCH-G-07 | GUIDE §1 | Durable run ledger + correlation | `sch_run` model, `/runs` | Implemented | `test_runs_list_get_cancel_and_invalid_cancel` | PASS |
| SCH-G-08 | GUIDE §1 | Cluster-safe tick locks / shards | `lock_manager.py`, `sch_tick_lock` | Implemented | `test_try_acquire_exclusive`, `test_tick_second_claimer_gets_lock_held` | PASS |
| SCH-G-09 | GUIDE §1 | Tenant-scoped schedules + quotas | `list_schedules`, `/quotas` | Implemented | `test_policies_handlers_quotas` | PASS |
| SCH-G-10 | GUIDE §2 | No business logic in ticker; UTC + explicit TZ | tick + cron parser | Implemented | `test_daily_8am_ist_next_fire` | PASS |
| SCH-G-11 | GUIDE §3 | Definition versioning; activate swaps | versions + activate APIs | Implemented | `test_versions_activate_pause_resume_retire`, `test_draft_only_mutable_and_handler_not_allowed` | PASS |
| SCH-G-12 | GUIDE §3 | Next-fire precompute; idempotent `(schedule_id, planned_fire_at)` | activations + planned unique | Implemented | `test_overlap_skip_and_unique_planned_fire` | PASS |
| SCH-G-13 | GUIDE §3 | Feature gates / pause cascades / dry-run | feature_flags, pause, dry-run tick | Implemented | `test_versions_activate_pause_resume_retire`, `test_tick_enqueue_and_dry_run` | PASS |
| SCH-G-14 | GUIDE §3 | Packs for system sweeps | seed packs + `/packages` | Implemented | `test_packages_health_stats_and_checksum` | PASS |
| SCH-G-15 | GUIDE §5–7 | Holiday/business calendars; permissions scheduler.* | calendars + `permissions/catalog.py` | Implemented | `test_calendars_and_blackout_crud`, `test_schedule_patch_invalid_cron_and_denied` | PASS |
| SCH-G-16 | GUIDE §9 | Domain events outbox | `sch_outbox`, OUTBOX_STREAM | Implemented | `test_sch_outbox_stream_constant` | PASS |
| SCH-G-17 | GUIDE §11 DoD | IST/DST golden tests; unique fire; overlap; lock; CATCH_UP_N; blackout; manual audit; RLS; no heavy ticker; no cross-schema FKs | tests + RLS migration `f17b` | Implemented | cron/misfire/overlap/lock/API suites | PASS |
| SCH-S-01 | SCHEMA §2 | 58 domain + outbox + idempotency = 60 `sch_*` tables | ORM models | Implemented | `test_sch_module_tables_count_60` | PASS |
| SCH-S-02 | SCHEMA §1 | Schema `scheduler` never `p17`; UTC timestamps | `schema_constants.py` | Implemented | table schema assert | PASS |
| SCH-S-03 | SCHEMA §3 | Enums kind/lifecycle/run/misfire/overlap/skip/calendar | `domain/enums.py` | Implemented | unit + API | PASS |
| SCH-S-04 | SCHEMA §4–10 | Schedule/version/specs/policies/calendars/runs/locks/bridges/deps | model modules + store | Implemented | module + API | PASS |
| SCH-S-05 | SCHEMA §11 | Seed packs messaging/event_bus/notify/cache | `seed_defaults` | Implemented | list schedules contains system keys | PASS |
| SCH-S-06 | SCHEMA §12–14 | Plumbing outbox/idempotency; RLS tenant; seed TZ + policies + allowlist + permissions | Alembic `f17a`/`f17b` | Implemented | module + permission catalog | PASS |
| SCH-A-01 | API §3–4 | Envelope + SCH_* errors | exception handlers | Implemented | 404/409/422/403 cases | PASS |
| SCH-A-02 | API §5 | Permissions catalog | `SCHEDULER_PERMISSIONS` | Implemented | denied 403 | PASS |
| SCH-A-03 | API §6 | CRUD schedules + versions + activate/pause/resume/retire | `/api/v1/scheduler/schedules*` | Implemented | catalog + version tests | PASS |
| SCH-A-04 | API §7 | Simulate next fires, no side effects | `/simulate`, `/versions/{id}/simulate` | Implemented | `test_simulate_next_fires_and_simulation_only` | PASS |
| SCH-A-05 | API §8 | Manual run-now + Idempotency-Key + audit | `/run-now` | Implemented | `test_run_now_idempotent_and_blackout` | PASS |
| SCH-A-06 | API §9 | Runs list/get/cancel | `/runs` | Implemented | `test_runs_list_get_cancel_and_invalid_cancel` | PASS |
| SCH-A-07 | API §10 | Calendars + blackouts | `/calendars`, `/blackouts` | Implemented | `test_calendars_and_blackout_crud` | PASS |
| SCH-A-08 | API §11 | Dependencies + cycle | `/dependencies` | Implemented | `test_dependencies_ok_and_cycle` | PASS |
| SCH-A-09 | API §12 | Policies + handler allowlist | `/policies/*`, `/handlers/allowlist` | Implemented | `test_policies_handlers_quotas` | PASS |
| SCH-A-10 | API §13 | Quotas + usage | `/quotas` | Implemented | `test_policies_handlers_quotas` | PASS |
| SCH-A-11 | API §14 | Internal tick / dry-run / heartbeat / rebalance | `/internal/v1/scheduler/*` | Implemented | tick + lock + rebalance tests | PASS |
| SCH-A-12 | API §15 | p14 callbacks started/succeeded/failed | internal run callbacks | Implemented | `test_tick_lock_heartbeat_rebalance_and_callbacks` | PASS |
| SCH-A-13 | API §16–17 | Packages, changesets, stats, tickers, health, audit | ops router | Implemented | `test_packages_health_stats_and_checksum` | PASS |
| SCH-A-14 | API §18 | Planned unique + manual idempotency | store uniqueness + idempotency map | Implemented | overlap + run-now tests | PASS |
| SCH-MOD | GUIDE §8 / registry | ModulePlugin `p17_scheduler` deps p01+p14; wired in main + alembic env | `SchedulerModule`, `apps/api/main.py`, `alembic/env.py` | Implemented | `test_sch_module_plugin_name_and_deps`, `test_sch_app_loads_p17_after_p16` | PASS |
