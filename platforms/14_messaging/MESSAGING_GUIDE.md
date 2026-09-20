# JeslotERP Messaging Platform — Developer Integration Guide

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — job enqueue/list/get persist on Postgres when `AsyncSession`; Redis/Rabbit broker is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p14_messaging`  
**PostgreSQL schema:** `messaging`  
**Depends on:** `p13_event_bus`  
**Integrates with:** `p01_identity`, `p03_configuration`, `p12_feature`, `p15_notification`, `p17_scheduler` (enqueues jobs), `p16_cache`, `p20_logging`, `p21_monitoring`, all platforms’ async workers  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`MESSAGING_SCHEMA.md`](MESSAGING_SCHEMA.md) · [`MESSAGING_API.md`](MESSAGING_API.md) · [`MESSAGING_RTM.md`](MESSAGING_RTM.md) · [`MESSAGING_IMPLEMENTATION_RECORD.md`](MESSAGING_IMPLEMENTATION_RECORD.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Live** | **2026-09-11** | Backend implementation Live: ORM 63 tables, in-memory job engine, HTTP APIs, Alembic, tests, RTM. |
| 1.2 | 2026-09-12 | TASK-SOR-025: HTTP job ledger Postgres-first (empty list is `[]`); broker port fail-closed; `POST /brokers/{key}/test-connection`. |
| **1.0 Advanced** | **2026-09-09** | Full enterprise job/queue plane: queues, priorities, delayed/FIFO jobs, workers/leases, retries/DLQ, idempotency, fair tenant scheduling, rate limits, batch jobs, handler registry, bridges from event-bus & scheduler. |

---

## 1. Purpose (enterprise)

`p14_messaging` is JeslotERP’s **async job & queue control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **Azure Service Bus / Amazon SQS / RabbitMQ** — queues, visibility, DLQ, delayed delivery  
- **SAP background processing / qRFC patterns** — reliable async work with retry discipline  
- **Salesforce Queueable / Batch Apex** — async work units with limits and monitoring  
- **Sidekiq / Celery / Hangfire class systems** — workers, priorities, schedules→jobs  

It is **not** Redis `LPUSH` from a request thread with no ledger. It is the system that makes ERP async work correct for:

1. **Durable job queues** with at-least-once execution  
2. **Priorities, delays, and FIFO groups** where needed  
3. **Worker pools** with leases / visibility timeouts / heartbeats  
4. **Retries with backoff** and **dead-letter queues**  
5. **Idempotent handlers** keyed per job  
6. **Fair multi-tenant scheduling** (noisy-neighbor control)  
7. **Rate limits** per queue/handler/tenant  
8. **Batch / chunk jobs** (large imports, reindex)  
9. **Bridges** — p13 deliveries → handler jobs; p17 cron → enqueue  
10. **Ops visibility** — lag, poison, redrive, pause/drain  

### Owns

| Domain | Examples |
|---|---|
| Queues | names, types, policies |
| Jobs / messages | payloads, state, attempts |
| Workers | registration, leases, heartbeats |
| Handlers | allow-listed handler keys |
| Retry / DLQ | policies, redrive |
| Delays / schedules-in-queue | `run_at` |
| FIFO / group ordering | session keys |
| Rate & fairness | tenant quotas |
| Batch jobs | parent/child chunks |
| Admin control | pause, purge, redrive |
| Bridges | event-bus & scheduler adapters |

### Does **not** own

| Concern | Owner |
|---|---|
| Domain event contracts / schema registry | `p13_event_bus` |
| Cron calendar definitions | `p17_scheduler` (enqueues here) |
| Email/SMS send semantics | `p15_notification` (uses jobs) |
| BPM human workflows | `p10_process` |
| Business transaction outbox | Platform outboxes → p13 → optional p14 job |

### Critical split: Messaging vs Event Bus vs Scheduler

| | **Messaging (p14)** | **Event Bus (p13)** | **Scheduler (p17)** |
|---|---|---|---|
| Unit | Job / command message | Domain CloudEvent | Cron / calendar trigger |
| Question | Do this work unit | Something happened | When to fire |
| Typical payload | `{handler, args}` | Business fact | Schedule id → enqueue job |

**Rule:** Prefer p13 for *facts*. Use p14 for *work*. Scheduler never does heavy work inline — it enqueues p14 jobs.

---

## 2. Architectural position

```text
API / Domain / p17 Scheduler / p13 Dispatcher
                    │
                    ▼
              enqueue job ──► queue (priority / delay / FIFO)
                    │
                    ▼
              worker lease ──► handler (allow-listed)
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        success   retry      DLQ → redrive
```

**Hard rules**

1. Handlers are **allow-listed** (`handler_key`) — no arbitrary import paths from payload.  
2. Jobs are durable in DB (or DB+broker with DB ledger); never memory-only in prod.  
3. At-least-once execution; handlers must be idempotent.  
4. No cross-schema FKs — payload UUIDs only.  
5. Tenant_id on jobs for RLS/fairness when business-scoped.  
6. Pause queue ≠ delete messages.

---

## 3. Advanced design principles

1. **Queue as policy container** — retry, rate, concurrency, fairness live on queue.  
2. **Job state machine** — clear transitions; no silent drops.  
3. **Lease/visibility** — exclusive processing window with heartbeat extension.  
4. **Priority bands** — CRITICAL / HIGH / NORMAL / LOW.  
5. **Delayed jobs** — `run_at` in future; invisible until due.  
6. **FIFO groups** — same `group_key` ordered; different groups parallel.  
7. **Idempotency keys** — enqueue dedupe window.  
8. **Poison detection** — consecutive handler crashes → quarantine.  
9. **Fair scheduling** — weighted round-robin across tenants.  
10. **Backpressure** — max depth / reject or shed low priority.  
11. **Batch parent jobs** — fan-out chunks; fan-in completion.  
12. **Dead letter ≠ delete** — inspectable + redrive.  
13. **Worker identity** — instance_id, version, capabilities.  
14. **Middleware** — timeout, metrics, tracing correlation.  
15. **Encrypted payloads optional** — secret_ref for keys; sensitive jobs flagged.  
16. **CQRS HTTP** — thin admin/enqueue APIs; hot path optimized.  
17. **Bridge from p13** — subscription handler_key may enqueue job envelope.  
18. **Packs** — seed system queues (default, mail, search, media, relay).

---

## 4. Core concepts

### 4.1 Queue kinds

| Kind | Use |
|---|---|
| `STANDARD` | Max throughput, unordered |
| `FIFO` | Ordered per `group_key` |
| `DELAY` | Emphasis on future `run_at` |
| `PRIORITY` | Multi-lane priority |
| `BATCH` | Parent/child orchestration |

### 4.2 Job lifecycle

```text
PENDING → DELAYED → READY → LEASED → RUNNING
                              │
                              ├─► SUCCEEDED
                              ├─► FAILED → (backoff) → READY
                              ├─► DEAD_LETTER
                              ├─► CANCELLED
                              └─► POISON
```

### 4.3 Job envelope

```text
job_id, queue_key, handler_key,
payload_json, tenant_id?, company_id?,
priority, run_at, group_key?,
idempotency_key?,
correlation_id, causation_id,
attempt, max_attempts,
timeout_ms, created_by_service
```

### 4.4 Worker lease

```text
lease_id, job_id, worker_id,
leased_at, lease_expires_at, heartbeat_at
```

Expired lease → job becomes READY again (attempt policy applies).

### 4.5 Handler registry

```text
handler_key = "media.scan_file"
module = "platforms.p08_file_media…"
timeout_ms, concurrency_hint, queue_default,
is_active, permission_to_enqueue
```

Code implements handler; DB registers metadata & allow-list.

---

## 5. Reliability model

| Guarantee | Notes |
|---|---|
| Enqueue | Durable after commit of messaging TX |
| Execute | At-least-once |
| Exactly-once effect | Handler idempotency + optional store |
| Delay accuracy | Near `run_at` (± poll interval) |
| FIFO | Per group_key; not global |

---

## 6. Fairness & rate limits

- Per-tenant max leased jobs / enqueue rate  
- Queue-level QPS for handlers  
- Shed `LOW` when depth > soft limit  
- Reject enqueue when hard depth exceeded (`MESSAGING_QUEUE_FULL`)

---

## 7. Integration patterns

### 7.1 Domain enqueues after commit

Prefer: domain outbox → p13 → consumer enqueues p14 job **or** domain calls messaging enqueue in same app service after TX (document carefully). Best: event-driven enqueue via p13 bridge for decoupling.

### 7.2 Scheduler

p17 fires → `POST /jobs` with handler `reports.generate_daily`.

### 7.3 Notification

p15 accepts command → enqueues `notify.dispatch_channel` jobs.

### 7.4 Media / search

Heavy work always via p14 (`media.variant`, `search.reindex_chunk`).

---

## 8. Security

### Permissions

| Code | Use |
|---|---|
| `messaging.queue.read` | Inspect queues |
| `messaging.queue.manage` | Create/update queues |
| `messaging.enqueue` | Enqueue jobs |
| `messaging.enqueue.{handler}` | Scoped enqueue |
| `messaging.worker` | Lease/ack (service role) |
| `messaging.dlq.manage` | Redrive/discard |
| `messaging.admin` | Pause/purge/rebalance |
| `messaging.audit.read` | Audit |
| `messaging.*` | Wildcard |

### RLS

FORCE RLS on tenant-scoped jobs when `tenant_id` set.  
System queues readable with permission.

---

## 9. Module layout

```text
platforms/p14_messaging/
  application/
    services/
      enqueue.py
      lease_manager.py
      dispatcher.py
      retry.py
      dlq.py
      fairness.py
      rate_limit.py
      batch_orchestrator.py
      fifo.py
      bridge_event_bus.py
    handlers/   # built-in system handlers only
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… workers/runner.py
    brokers/  # optional Redis/SB adapter behind ledger
  tests/unit/lease/ fifo/ retry/ fairness/
```

---

## 10. Domain events

| Event | When |
|---|---|
| `messaging.job.enqueued` / `succeeded` / `dead_lettered` | Job lifecycle |
| `messaging.queue.paused` / `resumed` | Ops |
| `messaging.queue.depth_high` | Alert |
| `messaging.worker.stale` | Ops |
| `messaging.dlq.redriven` | Ops |
| `messaging.batch.completed` | Batch |

Stream via p13 type registration when Live; local outbox until then.

---

## 11. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, queues, permissions |
| P2 | Enqueue + lease + succeed/fail |
| P3 | Retry/backoff + DLQ |
| P4 | Delayed jobs + priorities |
| P5 | FIFO groups |
| P6 | Fairness + rate limits |
| P7 | Batch parent/child |
| P8 | Event-bus & scheduler bridges |
| P9 | Packs (system queues) + admin UX APIs |
| P10 | Registry → **Live** |

---

## 12. Definition of Done (enterprise)

- [x] Lease expiry returns job to READY without loss  
- [x] Idempotent enqueue window works  
- [x] FIFO group ordering under concurrency  
- [x] Fairness prevents single-tenant starvation of others  
- [x] DLQ redrive restores to READY  
- [x] Unknown handler_key rejected  
- [x] Tenant RLS on job read APIs (`require_messaging_access` + `apply_messaging_rls`)  
- [x] Heartbeat extends lease  
- [x] No cross-schema FKs  
- [x] HTTP job list/get persist on `AsyncSession` (empty catalog is `[]`)  
- [x] Live Redis/Rabbit is a fail-closed port (`PROVIDER_PENDING`); pytest stays MEMORY + in-process runner  

---

## 13. Anti-patterns

| Don’t | Do |
|---|---|
| Run heavy work in HTTP request | Enqueue job |
| Arbitrary `import payload["module"]` | Allow-listed handler_key |
| Infinite retries | Max attempts + DLQ |
| One global lock for all jobs | Per-job lease / SKIP LOCKED |
| Put domain events only in p14 | Use p13 for facts |
| Cron does 10-minute work | Cron enqueues job |

---

## 14. Related documents

- Schema: [`MESSAGING_SCHEMA.md`](MESSAGING_SCHEMA.md)  
- API: [`MESSAGING_API.md`](MESSAGING_API.md)  
- Event bus: [`../13_event_bus/EVENT_BUS_GUIDE.md`](../13_event_bus/EVENT_BUS_GUIDE.md)  
- Scheduler: [`../17_scheduler/SCHEDULER_GUIDE.md`](../17_scheduler/SCHEDULER_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
