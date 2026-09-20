# JeslotERP Process Platform (p10) — Requirements Traceability Matrix

**Date:** 2026-09-10  
**Package:** `platforms.p10_process`  
**PostgreSQL schema:** `process`  
**Sources:** `PROCESS_GUIDE.md`, `PROCESS_SCHEMA.md`, `PROCESS_API.md`  
**Verification:** `pytest platforms/p10_process/tests -q` → **94 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| PROC-G-01 | GUIDE §1 | Workflow/approval control plane | `p10_process` + ProcessCatalogStore | Done | engine/API start | Pass |
| PROC-G-02 | GUIDE §2 | Domain starts; engine advances | start/complete services | Done | start + complete | Pass |
| PROC-G-03 | GUIDE §2 | No cross-schema FKs | UUID + business_key only | Done | ORM models | Pass |
| PROC-G-04 | GUIDE §3 | Definition versioning DRAFT→ACTIVE | publish/activate | Done | publish/pin tests | Pass |
| PROC-G-05 | GUIDE §3 | BPMN-lite nodes/edges | graph materialize | Done | XOR/parallel | Pass |
| PROC-G-06 | GUIDE §3 | Token execution | `_advance_token` | Done | parallel join | Pass |
| PROC-G-07 | GUIDE §3 | Work item state machine | claim/complete | Done | claim conflict | Pass |
| PROC-G-08 | GUIDE §3 | Agent strategies USER/ROLE/QUEUE | `_resolve_assignee` | Done | seeds + inbox | Pass |
| PROC-G-09 | GUIDE §3 | SLA clock + escalation | deadlines + tick | Done | escalation tests | Pass |
| PROC-G-10 | GUIDE §3 | Delegation OOO | put_delegation | Done | OOO apply/exclude | Pass |
| PROC-G-11 | GUIDE §3 | Unique active business_key | correlation | Done | duplicate 409 | Pass |
| PROC-G-12 | GUIDE §3 | Variables + strict allow-list path | variables map | Done | patch/redact | Pass |
| PROC-G-13 | GUIDE §3 | Service handler allow-list | handlers registry | Done | handlers API | Pass |
| PROC-G-14 | GUIDE §3 | Signals / message catch | signal + subscriptions | Done | unmatched + timer | Pass |
| PROC-G-15 | GUIDE §3 | Compensation on cancel | compensate path | Done | cancel compensate | Pass |
| PROC-G-16 | GUIDE §8 | Permissions `process.*` | permissions/catalog.py | Done | 403 gates | Pass |
| PROC-G-17 | GUIDE §9 | Module layout | `platforms/p10_process/**` | Done | module tests | Pass |
| PROC-G-18 | GUIDE §10 | Outbox `jesloterp:process:outbox` | ProcOutboxEvent + emit | Done | stream + health | Pass |
| PROC-S-01 | SCHEMA §2 | 63 domain tables | ORM split by section | Done | 66-table assert | Pass |
| PROC-S-02 | SCHEMA §2 | Plumbing outbox/idempotency/catalog_audit | 3 plumbing tables | Done | 66-table assert | Pass |
| PROC-S-03 | SCHEMA §1 | Schema name `process` never `p10` | `PROCESS_SCHEMA` | Done | schema constant | Pass |
| PROC-S-04 | SCHEMA §3 | Enumerations | `domain/enums.py` | Done | runtime flows | Pass |
| PROC-S-05 | SCHEMA §14 | Seeds outcomes/handlers/queues/processes | `seed_defaults` | Done | startup seeds | Pass |
| PROC-A-01 | API §5 | Permission codes | PROCESS_PERMISSIONS | Done | catalog + gate | Pass |
| PROC-A-02 | API §6 | Start/get/list/by-business/vars | `/instances/*` + `durable_start_instance` | Done | API + test_durable_sor | Pass |
| PROC-A-03 | API §6 | Cancel/suspend/resume/signal | instance ops | Done | API + engine | Pass |
| PROC-A-04 | API §7 | Inbox/claim/complete/reject | `/inbox` `/tasks/*` + `fetch_inbox` | Done | API + test_fetch_empty_instances_and_inbox_are_lists | Pass |
| PROC-SOR-01 | TASK-SOR-012 | HTTP instances/inbox on AsyncSession | runtime_repository + RLS access | Done | test_persist_instance_then_fetch_from_same_session | Pass |
| PROC-A-05 | API §7 | Reassign/delegate/comments/attachments | task routers | Done | API + reassign | Pass |
| PROC-A-06 | API §8 | Delegation me/admin | `/delegations` | Done | delegation API | Pass |
| PROC-A-07 | API §9 | Queues CRUD/members/inbox | `/queues` | Done | queue members | Pass |
| PROC-A-08 | API §10 | Processes/definitions/graph/publish | admin routers | Done | publish/activate | Pass |
| PROC-A-09 | API §10 | Simulate | `/definitions/{id}/simulate` | Done | simulate API | Pass |
| PROC-A-10 | API §11 | SLA policies + escalations + incidents | SLA/admin | Done | sla/escalation | Pass |
| PROC-A-11 | API §12 | Handlers registry | `/handlers` | Done | handlers API | Pass |
| PROC-A-12 | API §13 | Packages + changesets | packages/install | Done | checksum fail/ok | Pass |
| PROC-A-13 | API §14 | Migrations | migrate plan/execute | Done | forbidden/success | Pass |
| PROC-A-14 | API §15 | Audit + stats | audit/stats routes | Done | stats API | Pass |
| PROC-A-15 | API §16 | Internal ticks/start/signal/outcome/health | `/internal/v1/process` | Done | internal health/ticks | Pass |
| PROC-A-16 | API §4 | Error envelope | ProcessError handlers | Done | 403/409/422 | Pass |
| PROC-A-17 | API §1 | Idempotent start/complete | Idempotency-Key | Done | idempotent tests | Pass |
| PROC-W-01 | sample.md | RTM 100% | this file | Done | — | Pass |
| PROC-W-02 | sample.md | Implementation record 18 sections | `PROCESS_IMPLEMENTATION_RECORD.md` | Done | — | Pass |
| PROC-W-03 | Wiring | `apps/api/main.py` ModuleRegistry + exception handlers | apps/api/main.py | Implemented | test_load_modules_includes_process | Passed |
| PROC-W-04 | SCHEMA DDL/RLS | Alembic create + FORCE RLS | `c7d8e9f0a1b2` / `d8e9f0a1b2c3` | Implemented | alembic upgrade head | Passed |
| PROC-W-05 | alembic/env.py | Import p10 persistence models | alembic/env.py | Implemented | alembic heads | Passed |
| PROC-W-06 | Registry | PLATFORM_REGISTRY Live for p10 | PLATFORM_REGISTRY.md | Implemented | registry Live + checkbox | Passed |

---

**Coverage note:** All GUIDE/SCHEMA/API functional requirements mapped and verified under `platforms/p10_process` (**94** tests). HTTP instances + inbox are SoR-Live (not Production). Definitions/admin still use the in-memory catalog as a TestClient double.
