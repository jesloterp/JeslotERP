# JeslotERP Messaging Platform — Complete API Specification (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — job enqueue/list/get Postgres-first; broker `test-connection` is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p14_messaging`  
**PostgreSQL schema:** `messaging`  
**Public base:** `/api/v1/messaging`  
**Internal base:** `/internal/v1/messaging`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`MESSAGING_GUIDE.md`](MESSAGING_GUIDE.md) · [`MESSAGING_SCHEMA.md`](MESSAGING_SCHEMA.md) · [`MESSAGING_RTM.md`](MESSAGING_RTM.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Live** | **2026-09-11** | Implemented HTTP surface + in-memory engine; RTM/tests green. |
| 1.2 | 2026-09-12 | TASK-SOR-025: durable job ledger; `POST /brokers/{broker_key}/test-connection`. |
| **1.0 Advanced** | **2026-09-09** | Enqueue/lease/ack, delayed/FIFO/priority, DLQ/redrive, batch, fairness, bridges, admin pause/purge, worker APIs. |

---

## 1. Design principles (advanced)

1. **Enqueue + worker lease are primary** — business services enqueue; workers claim.  
2. **Allow-listed handlers only** — unknown `handler_key` → reject.  
3. **At-least-once** — clients design idempotent handlers.  
4. **Idempotent enqueue** — `Idempotency-Key` / job `idempotency_key`.  
5. **Lease + heartbeat** — no orphan RUNNING forever.  
6. **DLQ visible** — max attempts never silent-drop.  
7. **Pause ≠ purge** — admin control is explicit.  
8. **FIFO is per group_key** — not global ordering.  
9. **Fairness enforced** on claim, not only enqueue.  
10. **Internal worker API** separate from human admin API.  
11. **Payload size capped**; large blobs via media_id.  
12. **Sensitive payloads** redacted in list endpoints.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Correlation-Id: <uuid>
X-Worker-Key: <instance-id>    # worker calls
X-Internal-Token: <token>
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": { "request_id": "…" }
}
```

---

## 4. Errors

```text
MSG_QUEUE_NOT_FOUND / QUEUE_PAUSED / QUEUE_FULL
MSG_HANDLER_UNKNOWN / HANDLER_INACTIVE / PAYLOAD_INVALID
MSG_JOB_NOT_FOUND / INVALID_STATUS
MSG_LEASE_NOT_FOUND / LEASE_EXPIRED / LEASE_OWNED_BY_OTHER
MSG_IDEMPOTENCY_CONFLICT
MSG_FIFO_BLOCKED / GROUP_INFLIGHT
MSG_RATE_LIMITED / TENANT_QUOTA_EXCEEDED
MSG_BATCH_NOT_FOUND / BATCH_INVALID
MSG_DLQ_EMPTY / REDRIVE_CONFLICT
MSG_WORKER_UNAUTHORIZED / WORKER_STALE
MSG_CANCEL_CONFLICT
MSG_DEPTH_SHED
MSG_PACKAGE_CHECKSUM_MISMATCH
MSG_VERSION_CONFLICT
```

HTTP: `404` · `409` · `422` · `403` · `429` · `503` (paused/full).

---

## 5. Permissions

| Code | Use |
|---|---|
| `messaging.queue.read` | Inspect |
| `messaging.queue.manage` | Mutate queues/policies |
| `messaging.enqueue` | Enqueue |
| `messaging.worker` | Lease/complete (services) |
| `messaging.dlq.manage` | Redrive |
| `messaging.admin` | Pause/purge |
| `messaging.audit.read` | Audit |
| `messaging.*` | All |

---

## 6. Enqueue APIs (primary for producers)

### 6.1 Enqueue job

```http
POST /api/v1/messaging/jobs
Idempotency-Key: …
```

```json
{
  "handler_key": "media.scan_file",
  "queue_key": "media",
  "payload": { "media_id": "…" },
  "tenant_id": "…",
  "company_id": "…",
  "priority": "HIGH",
  "run_at": null,
  "group_key": null,
  "idempotency_key": "scan:media:…",
  "timeout_ms": 120000,
  "max_attempts": 5,
  "links": [
    { "entity_type": "media.object", "entity_id": "…" }
  ]
}
```

**Response:**

```json
{
  "job_id": "…",
  "status": "READY",
  "queue_key": "media",
  "run_at": "2026-09-09T04:27:00Z",
  "idempotent_replay": false
}
```

If `run_at` future → `DELAYED`. Unknown handler → `422 MSG_HANDLER_UNKNOWN`.

### 6.2 Enqueue batch (fan-out)

```http
POST /api/v1/messaging/batches
Idempotency-Key: …
```

```json
{
  "batch_key": "reindex-tenant-…",
  "handler_key": "search.reindex_chunk",
  "queue_key": "search",
  "tenant_id": "…",
  "chunks": [
    { "payload": { "from_id": "…", "to_id": "…" } },
    { "payload": { "from_id": "…", "to_id": "…" } }
  ]
}
```

Creates parent batch + child jobs.

### 6.3 Get job / list

```http
GET /api/v1/messaging/jobs/{job_id}
GET /api/v1/messaging/jobs?queue_key=media&status=DEAD_LETTER&tenant_id=…
GET /api/v1/messaging/batches/{batch_id}
```

### 6.4 Cancel

```http
POST /api/v1/messaging/jobs/{job_id}/cancel
POST /api/v1/messaging/batches/{batch_id}/cancel
```

Cancels if not terminal; RUNNING may be cooperative (cancel flag checked by handler).

---

## 7. Worker APIs (internal / primary for executors)

### 7.1 Register / heartbeat worker

```http
POST /internal/v1/messaging/workers/register
POST /internal/v1/messaging/workers/{worker_key}/heartbeat
```

```json
{
  "worker_key": "media-worker-1",
  "version": "1.4.0",
  "max_leases": 8,
  "capabilities": { "queues": ["media"], "handlers": ["media.*"] }
}
```

### 7.2 Claim / lease jobs

```http
POST /internal/v1/messaging/queues/{queue_key}/claim
```

```json
{
  "worker_key": "media-worker-1",
  "max_jobs": 5,
  "visibility_timeout_seconds": 60
}
```

**Response:** jobs with payload + `lease_id` + `lease_expires_at`.  
Respects pause, fairness, FIFO inflight, rate limits.

### 7.3 Heartbeat lease

```http
POST /internal/v1/messaging/leases/{lease_id}/heartbeat
```

Extends expiry; fails if expired/owned by other → `MSG_LEASE_EXPIRED`.

### 7.4 Complete / fail

```http
POST /internal/v1/messaging/leases/{lease_id}/succeed
POST /internal/v1/messaging/leases/{lease_id}/fail
```

**Succeed:**

```json
{ "result": { "ok": true, "digest": "…" } }
```

**Fail:**

```json
{
  "error_code": "SCAN_ENGINE_TIMEOUT",
  "error_detail": "…",
  "retryable": true
}
```

Retryable → schedule next attempt with backoff; else or max attempts → DLQ.

### 7.5 Release (without complete)

```http
POST /internal/v1/messaging/leases/{lease_id}/release
```

Returns job to READY without consuming attempt (policy-dependent; default consumes soft attempt for abuse control).

---

## 8. Delayed / FIFO / priority helpers

```http
POST /api/v1/messaging/jobs/{job_id}/reschedule
```

```json
{ "run_at": "2026-09-09T10:00:00Z", "priority": "NORMAL" }
```

Only when status DELAYED/READY/FAILED(waiting).

FIFO: set `group_key` on enqueue; claim automatically serializes per group.

---

## 9. DLQ & redrive

```http
GET  /api/v1/messaging/queues/{queue_key}/dlq
POST /api/v1/messaging/queues/{queue_key}/dlq/redrive
POST /api/v1/messaging/dlq/{dlq_id}/discard
GET  /api/v1/messaging/redrive-jobs/{id}
```

**Redrive:**

```json
{
  "job_ids": ["…"],
  "reset_attempts": true,
  "run_at": null
}
```

---

## 10. Queue admin

```http
GET    /api/v1/messaging/queues
POST   /api/v1/messaging/queues
GET    /api/v1/messaging/queues/{queue_key}
PATCH  /api/v1/messaging/queues/{queue_key}
PUT    /api/v1/messaging/queues/{queue_key}/policy
POST   /api/v1/messaging/queues/{queue_key}/pause
POST   /api/v1/messaging/queues/{queue_key}/resume
POST   /api/v1/messaging/queues/{queue_key}/drain
GET    /api/v1/messaging/queues/{queue_key}/metrics
POST   /api/v1/messaging/queues/{queue_key}/purge-ready   # dangerous; admin + confirm
```

**Pause body:** `{ "reason": "Incident" }` — claim returns empty; enqueue may still accept unless `reject_on_pause`.

**Drain:** finish inflight; reject new enqueue optional.

**Purge:** deletes/cancels READY/DELAYED — requires typed confirm token.

---

## 11. Handlers registry

```http
GET  /api/v1/messaging/handlers
POST /api/v1/messaging/handlers
PATCH /api/v1/messaging/handlers/{handler_key}
POST /api/v1/messaging/handlers/{handler_key}/deactivate
POST /api/v1/messaging/handlers/{handler_key}/validate-payload
```

Validate-payload checks JSON schema without enqueue.

---

## 12. Fairness & quotas

```http
GET /api/v1/messaging/quotas
PUT /api/v1/messaging/quotas/tenants/{tenant_id}
GET /api/v1/messaging/quotas/tenants/{tenant_id}/usage
```

---

## 13. Bridges

```http
GET  /api/v1/messaging/bridges/event-bus
POST /api/v1/messaging/bridges/event-bus
GET  /api/v1/messaging/bridges/scheduler
POST /api/v1/messaging/bridges/scheduler
```

**Event-bus bridge:**

```json
{
  "event_subscription_key": "search.index.document.released",
  "handler_key": "search.index_document",
  "queue_key": "search",
  "payload_map": {
    "document_id": "$.data.document_id",
    "version_id": "$.data.version_id"
  }
}
```

Internal:

```http
POST /internal/v1/messaging/bridges/event-bus/ingest
```

Called by p13 worker dispatcher with CloudEvent → enqueue.

---

## 14. Sweeper ticks (internal)

```http
POST /internal/v1/messaging/sweep/leases
POST /internal/v1/messaging/sweep/delayed
POST /internal/v1/messaging/sweep/workers
POST /internal/v1/messaging/metrics/sample
```

- leases: expire → READY/FAIL  
- delayed: DELAYED due → READY  
- workers: mark STALE  

Usually driven by p17 or embedded worker loop.

---

## 15. Packages & governance

```http
GET  /api/v1/messaging/packages
POST /api/v1/messaging/packages/{package_key}/install
GET  /api/v1/messaging/changesets
POST /api/v1/messaging/changesets
POST /api/v1/messaging/changesets/{id}/approvals
```

---

## 16. Audit

```http
GET /api/v1/messaging/jobs/{job_id}/attempts
GET /api/v1/messaging/admin-audit?queue_key=…&from=…
```

---

## 17. Health

```http
GET /api/v1/messaging/health
GET /internal/v1/messaging/health
POST /api/v1/messaging/brokers/{broker_key}/test-connection
```

Reports queue depths, stale workers, DLQ counts. `broker` is `PROVIDER_PENDING`; `broker_kind` is `MEMORY` under pytest.

Broker `test-connection`: `memory` / `stub` / `dev` → MEMORY `OK` (in-process runner). `redis` / `rabbit` / `servicebus` → `ok: false`, `status: PROVIDER_PENDING`. Requires `messaging.admin` via `require_messaging_access`. No live Redis/Rabbit client in this slice.

Job enqueue/list/get persist on `AsyncSession`; empty list is `[]` (not the in-memory seed). TestClient `AsyncMock` still uses `MessagingCatalogStore`.

---

## 18. Caching & concurrency

| Resource | Strategy |
|---|---|
| Queue policies | Cached; invalidate on patch |
| Claim | `SKIP LOCKED` + fairness picker |
| Idempotent enqueue | Unique + return existing job |
| Lease heartbeat | Conditional update on owner |
| FIFO | Single inflight row lock per group |

---

## 19. Example flows

### 19.1 Media scan

1. p08 complete upload → enqueue `media.scan_file`  
2. Media worker claims → scans → succeed/fail  
3. On success domain continues via callback/event  

### 19.2 Delayed notify

1. Enqueue `notify.dispatch_channel` with `run_at=+15m`  
2. Sweeper promotes to READY  
3. Notify worker sends  

### 19.3 FIFO per sales order

1. Jobs with `group_key=sales.order:{id}` for sequential post-steps  
2. Claim ensures one inflight per group  

### 19.4 Incident

1. Pause `search` queue  
2. Fix handler  
3. Redrive DLQ  
4. Resume  

---

## 20. Event hooks

| Event | Consumer |
|---|---|
| `messaging.queue.depth_high` | Autoscale workers |
| `messaging.job.dead_lettered` | On-call |
| `messaging.worker.stale` | Ops |
| `messaging.batch.completed` | Domain continuation |

---

## 21. Compatibility notes

- Public prefix `/api/v1/messaging`; schema `messaging`.  
- p13 owns event meaning; p14 owns job execution.  
- p17 should enqueue, not run long work.  
- Handler code lives in owning platforms; messaging only dispatches by key.

---

## 22. Related documents

- Guide: [`MESSAGING_GUIDE.md`](MESSAGING_GUIDE.md)  
- Schema: [`MESSAGING_SCHEMA.md`](MESSAGING_SCHEMA.md)  
- Event bus: [`../13_event_bus/EVENT_BUS_API.md`](../13_event_bus/EVENT_BUS_API.md)  
- Scheduler: [`../17_scheduler/SCHEDULER_API.md`](../17_scheduler/SCHEDULER_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
