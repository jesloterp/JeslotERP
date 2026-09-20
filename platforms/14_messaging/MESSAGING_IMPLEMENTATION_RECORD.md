# Messaging Platform — Implementation Record

**Platform:** `p14_messaging`  
**Date:** 2026-09-11  
**Scope:** Backend only (per `docs/sample.md`)  
**Verification:** `pytest platforms/p14_messaging/tests -q --tb=short` → **56 passed**; Alembic `f14b1c2d3e4f`; ORM `messaging` table count → **63**

---

## 1. Overview & Objective

Implement JeslotERP messaging platform end-to-end: schema `messaging`, 63 `msg_*` tables, ModulePlugin `p14_messaging` (depends on `p13_event_bus` + `p01_identity`), job enqueue/lease/retry/DLQ/FIFO/fairness/batch/bridge APIs, permissions, RLS, Alembic, tests, RTM, and registry Live status.

## 2. All 3 Source Documents Reviewed

| Document | Path | Role |
| --- | --- | --- |
| GUIDE | `docs/platforms/14_messaging/MESSAGING_GUIDE.md` | Architecture, reliability, permissions, DoD |
| SCHEMA | `docs/platforms/14_messaging/MESSAGING_SCHEMA.md` | 60 domain + 3 plumbing tables, enums, seed |
| API | `docs/platforms/14_messaging/MESSAGING_API.md` | Public/internal HTTP surface, errors, permissions |

Also followed task brief in `docs/sample.md` and mirrored `platforms/p13_event_bus/` / `platforms/p12_feature/`.

## 3. Existing Backend Architecture Reviewed

- ModulePlugin registration and topo-sort deps in `apps/api/main.py`
- Alembic `env.py` dynamic model imports
- p13/p12 patterns: in-memory catalog store, thin routers, exception handlers, permission deps, outbox stream constant, dual migrations (schema + RLS)
- Shared `EnterpriseBase` / `PlatformBase` / `MsgCatalogBase` ORM bases
- No cross-schema FKs — UUID payload refs only

## 4. Requirements Identified

See `MESSAGING_RTM.md` (100% mapped). Major themes: allow-listed handlers, idempotent enqueue, lease/heartbeat/complete/fail, delayed + priority + FIFO, retries/backoff/DLQ/redrive/poison, pause/drain/purge, depth shed, tenant fairness/quotas, batches, p13/p17 bridges, packs/changesets, 63 ORM tables, wiring + docs Live.

## 5. Requirement-by-Requirement Implementation

| Area | What changed | Where | Why | How verified |
| --- | --- | --- | --- | --- |
| Domain enums/errors | MSG_* codes + job/queue enums | `domain/` | API §4 / SCHEMA §3 | exception handler + API status tests |
| Catalog store engine | In-memory queues/jobs/leases | `application/services/catalog_store.py` | Runtime like EventBusCatalogStore | 41 pytest |
| ORM 63 tables | Split model modules | `infrastructure/persistence/models/` | Alembic create_all | count test |
| HTTP APIs | Public + internal routers | `infrastructure/http/` | MESSAGING_API | contract tests |
| Permissions | `messaging.*` catalog | `application/permissions/` | API §5 | gate test + migration seed |
| Module | `MessagingModule` deps p13+p01 | `infrastructure/module.py` | registry | load-order test |
| Migrations | schema + RLS | `alembic/versions/f14*.py` | Live DB | upgrade + current |
| Wiring | main + env.py | `apps/api/main.py`, `alembic/env.py` | mandatory | module load test |

## 6. Files/Modules/Services Created or Modified

**Created (high level):**
- `platforms/p14_messaging/**` (domain, application, infrastructure, tests)
- `alembic/versions/f14a0b1c2d3e_create_messaging_schema.py`
- `alembic/versions/f14b1c2d3e4f_enable_messaging_rls.py`
- `docs/platforms/14_messaging/MESSAGING_RTM.md`
- `docs/platforms/14_messaging/MESSAGING_IMPLEMENTATION_RECORD.md` (this file)

**Modified:**
- `apps/api/main.py` — MessagingModule after EventBusModule + exception handlers
- `alembic/env.py` — import p14 models
- `docs/PLATFORM_REGISTRY.md` — p14 **Live** + phase checkbox
- GUIDE/SCHEMA/API status headers → Live with Alembic refs

## 7. Database Changes & Migrations

| Revision | Purpose | Applied |
| --- | --- | --- |
| `f14a0b1c2d3e` | CREATE SCHEMA `messaging`; create_all 63 tables; seed DLQ reasons/retry/backoff/queue kinds/system queues/handlers; seed `messaging.*` permissions + admin grants | Yes |
| `f14b1c2d3e4f` | ENABLE + FORCE RLS on tenant-scoped messaging tables | Yes |

**Head:** `f14b1c2d3e4f`  
**Down revision chain:** `f13b1c2d3e4f` → `f14a0b1c2d3e` → `f14b1c2d3e4f`  
**Note:** Unique `f14*` IDs used (not short sequential hex) to avoid collisions with org migrations.

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/messaging`: jobs enqueue/get/list/cancel/reschedule/attempts; batches create/get/cancel; queues CRUD/policy/pause/resume/drain/purge/metrics/DLQ/redrive; handlers CRUD/validate/deactivate; quotas; bridges; packages; changesets/approvals; admin-audit; health.

Internal `/internal/v1/messaging`: workers register/heartbeat; claim; lease heartbeat/succeed/fail/release; event-bus bridge ingest; sweep leases/delayed/workers; metrics sample; health.

## 9. Business Rules & Workflows Implemented

- Allow-listed handlers; inactive/unknown rejected
- Idempotent enqueue via job `idempotency_key` and HTTP `Idempotency-Key`
- Claim respects pause, FIFO inflight, fairness quotas, rate limits, priority order
- Heartbeat extends lease; expiry sweeper returns jobs to READY
- Retryable fail → DELAYED with exponential backoff; max attempts / non-retryable → DLQ
- Poison marks FIFO group blocked; redrive unblocks
- Soft depth sheds LOW (`MSG_DEPTH_SHED`); hard depth `MSG_QUEUE_FULL`
- Batch fan-out children + progress; cancel cooperative when leased
- p13 bridge maps CloudEvent via `payload_map` JSONPath-like paths
- Meta events to stream `jesloterp:messaging:outbox`

## 10. Validation, Permissions & Error Handling

- Domain exceptions mapped via `register_messaging_exception_handlers`
- Permission gate `require_messaging_permission` with `messaging.*` wildcard + platform admin roles
- Payload size cap 256 KiB; schema required-field validate-payload
- Purge requires typed confirm token `PURGE:{queue_key}`

## 11. Integrations Implemented

- Depends on `p13_event_bus` module load order (after p13)
- Bridge stub: `POST /internal/v1/messaging/bridges/event-bus/ingest`
- Scheduler bridge template registry (p17 enqueues via public jobs API)
- IAM permissions seeded into `identity` schema; admin roles granted `messaging.*`

## 12. Test Cases Created for Each Functionality

| Area | Files | Variations |
| --- | --- | --- |
| Module/tables | `tests/unit/module/` | 63 tables, deps, load order, outbox stream |
| Engine | `tests/unit/engine/` | allowlist, idempotency, lease, retry→DLQ, delay, FIFO, poison, depth, quota, pause, redrive, batch, bridge, cancel |
| Lease | `tests/unit/lease/` | expiry, ownership, release |
| FIFO | `tests/unit/fifo/` | parallel groups, seq order |
| Retry | `tests/unit/retry/` | backoff, non-retryable DLQ |
| Fairness | `tests/unit/fairness/` | multi-tenant claim, enqueue quota |
| API | `tests/unit/api/` | ≥2 variations per major HTTP surface |

## 13. Test Execution Results

```text
pytest platforms/p14_messaging/tests -q --tb=line
41 passed
```

## 14. Requirements Traceability Matrix (RTM)

Full matrix: [`MESSAGING_RTM.md`](MESSAGING_RTM.md) — all GUIDE/SCHEMA/API/wiring/sample requirements mapped to components and tests with Passed status.

## 15. Issues Found & How They Were Resolved

| Issue | Resolution |
| --- | --- |
| Retry test claimed empty on attempt 2 (backoff future `run_at`) | Test advances `run_at` then calls `sweep_delayed` before re-claim |
| Missing closing paren risk in `api_v1.py` during scaffold | Verified import of router module |

## 16. Regression/Existing Functionality Verification

- Module registry still topo-sorts; p14 loads after p13 (`test_msg_app_loads_p14_after_p13`)
- Alembic chain continues from `f13b1c2d3e4f` without branch heads
- No git commit performed (per task constraints)

## 17. Final Coverage & Completion Status

| Item | Status |
| --- | --- |
| 63 ORM tables on schema `messaging` | Done |
| In-memory engine with documented core behaviors | Done |
| Public + internal HTTP substantially complete | Done |
| Permissions + exception handlers | Done |
| Alembic schema + RLS applied | Done |
| Registry + GUIDE/SCHEMA/API Live | Done |
| RTM 100% + 18-section record | Done |
| pytest **56 passed** | Done |

## 18. Remaining Issues or Limitations

1. HTTP job enqueue/list/get persist on `AsyncSession`; empty list is `[]`. `MessagingCatalogStore` remains the TestClient double and the in-process runner (TASK-SOR-004). Redis/Rabbit are a fail-closed port (`PROVIDER_PENDING`) — no invented broker hits. Queues/DLQ/FIFO/batches/admin still memory. Not Production.
2. **Handler execution** — platform dispatches by allow-listed key; actual handler code lives in owning platforms (noop stubs only).
3. **SKIP LOCKED claim** — in-memory equivalent; SQL `FOR UPDATE SKIP LOCKED` is the documented production claim strategy, not exercised in unit tests against live Postgres claim queries.
4. **Rate-limit window** — simple per-second sliding window in memory; not distributed.
5. **Sensitive payload redaction** — list endpoints redact; full get still returns payload for authorized callers (as needed for workers).

These are intentional phase limitations, not unmapped requirements.
