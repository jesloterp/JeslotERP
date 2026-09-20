# Messaging Platform — Requirements Traceability Matrix (RTM)

**Platform:** `p14_messaging` · **Schema:** `messaging`  
**Sources:** MESSAGING_GUIDE.md · MESSAGING_SCHEMA.md · MESSAGING_API.md · docs/sample.md  
**Verification date:** 2026-09-11  
**Test command:** `pytest platforms/p14_messaging/tests -q --tb=short` → **56 passed**  
**Alembic:** `f14a0b1c2d3e` → `f14b1c2d3e4f (head)`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| MSG-G-01 | GUIDE §1 | Async job & queue control plane | `platforms/p14_messaging` ModulePlugin | Done | `test_msg_module_*` | Passed |
| MSG-G-02 | GUIDE §2 | Enqueue → lease → succeed/fail/DLQ | `MessagingCatalogStore` | Done | engine + API worker tests | Passed |
| MSG-G-03 | GUIDE §2 | Allow-listed `handler_key` only | `enqueue` / handlers registry | Done | `test_msg_enqueue_allowlist_*`, API unknown handler | Passed |
| MSG-G-04 | GUIDE §2 | No cross-schema FKs | UUID refs in payload only | Done | ORM review | Passed |
| MSG-G-05 | GUIDE §3 | Queue as policy container | queue + policy store/APIs | Done | queue create/policy/metrics tests | Passed |
| MSG-G-06 | GUIDE §3 | Job state machine | `MsgJobStatus` transitions | Done | lease/retry/cancel/dlq tests | Passed |
| MSG-G-07 | GUIDE §3 | Lease + heartbeat visibility | claim/heartbeat/succeed/fail | Done | `test_msg_lease_*`, API claim | Passed |
| MSG-G-08 | GUIDE §3 | Priority bands | `PRIORITY_SCORES` + claim sort | Done | depth shed LOW + reschedule priority | Passed |
| MSG-G-09 | GUIDE §3 | Delayed jobs `run_at` | DELAYED + sweep_delayed | Done | `test_msg_delayed_*`, reschedule API | Passed |
| MSG-G-10 | GUIDE §3 | FIFO per `group_key` | fifo_groups + claim gate | Done | `test_msg_fifo_*` | Passed |
| MSG-G-11 | GUIDE §3 | Idempotent enqueue | idempotency map + Idempotency-Key | Done | engine + API idempotency tests | Passed |
| MSG-G-12 | GUIDE §3 | Poison → quarantine / block group | `_to_poison` | Done | `test_msg_fifo_poison_blocks_group` | Passed |
| MSG-G-13 | GUIDE §3/§6 | Fairness / tenant quotas | quotas on enqueue/claim | Done | `test_msg_fairness_*`, quota API | Passed |
| MSG-G-14 | GUIDE §3/§6 | Backpressure soft/hard depth | QUEUE_FULL / DEPTH_SHED | Done | `test_msg_depth_shed_and_queue_full` | Passed |
| MSG-G-15 | GUIDE §3 | Batch parent/child | create_batch / progress | Done | engine + API batch tests | Passed |
| MSG-G-16 | GUIDE §3 | DLQ inspectable + redrive | dlq/redrive/discard | Done | engine + API dlq tests | Passed |
| MSG-G-17 | GUIDE §3 | Worker identity | register/heartbeat | Done | API worker register + lease tests | Passed |
| MSG-G-18 | GUIDE §3 | Bridge p13 → job | bridges + ingest | Done | `test_msg_bridge_*`, API bridges | Passed |
| MSG-G-19 | GUIDE §3 | Packs seed system queues | packages + seed_defaults | Done | packages install + module seeds | Passed |
| MSG-G-20 | GUIDE §8 | Permissions `messaging.*` | permissions catalog + gates | Done | pause permission denied | Passed |
| MSG-G-21 | GUIDE §8 | RLS FORCE tenant tables | Alembic `f14b1c2d3e4f` | Done | migration applied | Passed (DB) |
| MSG-G-22 | GUIDE §10 | Meta outbox `jesloterp:messaging:outbox` | `OUTBOX_STREAM` + `msg_outbox` | Done | health + module constant | Passed |
| MSG-G-23 | GUIDE §12 | DoD: lease expiry, FIFO, DLQ, unknown handler | engine | Done | lease/fifo/dlq/allowlist tests | Passed |
| MSG-S-01 | SCHEMA §2 | 60 domain + 3 plumbing = 63 tables | ORM models | Done | `test_msg_module_tables_count_63` | Passed |
| MSG-S-02 | SCHEMA §1 | Schema name `messaging` never p14 | `MESSAGING_SCHEMA` | Done | module/tables tests | Passed |
| MSG-S-03 | SCHEMA §15 | Seed queues/handlers/retry/DLQ reasons/permissions | store + Alembic | Done | list queues/handlers + migration | Passed |
| MSG-A-01 | API §4 | Error codes MSG_* | `domain/exceptions.py` + handlers | Done | 403/404/409/422/503 API tests | Passed |
| MSG-A-02 | API §5 | Permission codes | `MESSAGING_PERMISSIONS` + migration | Done | permission denied test | Passed |
| MSG-A-03 | API §6 | Enqueue/get/list/cancel/batch | jobs + admin routers | Done | API jobs/batches tests | Passed |
| MSG-A-04 | API §7 | Worker register/claim/heartbeat/complete/fail/release | internal router | Done | API worker claim tests | Passed |
| MSG-A-05 | API §8 | Reschedule delayed/ready | jobs router | Done | admin_audit/reschedule test | Passed |
| MSG-A-06 | API §9 | DLQ list/redrive/discard | queues router | Done | dlq API tests | Passed |
| MSG-A-07 | API §10 | Queue CRUD/pause/resume/drain/purge/metrics | queues router | Done | pause/create/purge/metrics | Passed |
| MSG-A-08 | API §11 | Handlers registry + validate-payload | admin router | Done | handlers API test | Passed |
| MSG-A-09 | API §12 | Quotas + usage | admin router | Done | quotas API test | Passed |
| MSG-A-10 | API §13 | Bridges event-bus/scheduler + ingest | admin + internal | Done | bridges API test | Passed |
| MSG-A-11 | API §14 | Sweep leases/delayed/workers + metrics sample | internal router | Done | sweep in health/sweep test | Passed |
| MSG-A-12 | API §15 | Packages + changesets/approvals | admin router | Done | packages/changesets test | Passed |
| MSG-A-13 | API §16 | Attempts + admin-audit | jobs + admin | Done | attempts via engine; audit API | Passed |
| MSG-A-14 | API §17 | Health public + internal | admin + internal | Done | health test | Passed |
| MSG-W-01 | Wiring | main.py MessagingModule after EventBus + handlers | `apps/api/main.py` | Done | `test_msg_app_loads_p14_after_p13` | Passed |
| MSG-W-02 | Wiring | alembic env import models | `alembic/env.py` | Done | upgrade applied | Passed |
| MSG-W-03 | Wiring | Migrations after event_bus head | `f14a0b1c2d3e` → `f14b1c2d3e4f` | Done | `alembic current` | Passed |
| MSG-T-01 | sample.md | ≥2 variations per major area | api/engine/lease/fifo/retry/fairness/module | Done | suite | Passed |
| MSG-SOR-25 | TASK-SOR-025 | Durable PG job ledger + broker port | durable.py + job_repository + BrokerPort | Implemented | `test_durable_sor` + `test_broker_port` | PASS |

**Coverage note:** HTTP job enqueue/list/get persist on `AsyncSession`; empty list is `[]`. `MessagingCatalogStore` is the TestClient double. In-process runner (TASK-SOR-004) stays MEMORY. Redis/Rabbit are `PROVIDER_PENDING`. Queues/DLQ/FIFO/batches stay memory.
