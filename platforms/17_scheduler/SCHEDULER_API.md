# JeslotERP Scheduler Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — calendars/blackouts + tickers Postgres-first when session is AsyncSession. Not Production.  
**Package:** `platforms.p17_scheduler`  
**PostgreSQL schema:** `scheduler`  
**Public base:** `/api/v1/scheduler`  
**Internal base:** `/internal/v1/scheduler`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`SCHEDULER_GUIDE.md`](SCHEDULER_GUIDE.md) · [`SCHEDULER_SCHEMA.md`](SCHEDULER_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Schedules CRUD/activate, simulate next fires, manual run, pause, calendars/blackouts, runs ledger, tick engine internals, packs. |

---

## 1. Design principles (advanced)

1. **Enqueue-only side effects** — tick creates run + p14 job; no heavy inline work.  
2. **Timezone explicit** on every recurring definition.  
3. **Idempotent fires** — unique planned fire instant.  
4. **Manual run audited** and rate-limited.  
5. **Overlap/misfire always declared**.  
6. **Simulate before activate** for cron changes.  
7. **Production critical schedules** may require approval.  
8. **Tenant isolation** on list/trigger.  
9. **Cluster-safe tick** via locks/shards.  
10. **Callbacks** from messaging update run status.  
11. **Blackouts** default suppress non-critical.  
12. **Dry-run tick** for ops (optional).

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
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
SCH_SCHEDULE_NOT_FOUND / VERSION_NOT_FOUND / NOT_ACTIVE
SCH_CRON_INVALID / TIMEZONE_INVALID / KIND_INVALID
SCH_OVERLAP_CONFLICT / BLACKOUT_ACTIVE / FEATURE_OFF
SCH_DEPENDENCY_UNMET / DEPENDENCY_CYCLE
SCH_HANDLER_NOT_ALLOWED / BRIDGE_INVALID
SCH_MANUAL_DENIED / RATE_LIMITED / QUOTA_EXCEEDED
SCH_RUN_NOT_FOUND / INVALID_STATUS
SCH_TICK_LOCK_HELD / SHARD_UNAVAILABLE
SCH_CATCHUP_BOUNDED
SCH_APPROVAL_REQUIRED / PACKAGE_CHECKSUM_MISMATCH
SCH_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
SCH_SIMULATION_ONLY
```

HTTP: `404` · `409` · `422` · `403` · `429` · `503`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `scheduler.catalog.read` | Read |
| `scheduler.catalog.manage` | Mutate definitions |
| `scheduler.run` | Manual trigger |
| `scheduler.pause` | Pause/resume |
| `scheduler.admin` | Blackouts/tickers |
| `scheduler.audit.read` | Runs/audit |
| `scheduler.pack.install` | Packs |
| `scheduler.*` | All |

---

## 6. Schedule catalog

### 6.1 List / create

```http
GET  /api/v1/scheduler/schedules
POST /api/v1/scheduler/schedules
GET  /api/v1/scheduler/schedules/{schedule_key}
PATCH /api/v1/scheduler/schedules/{schedule_key}
```

**Create:**

```json
{
  "schedule_key": "notify.digests.flush.daily",
  "name": "Flush notification digests",
  "kind": "CRON",
  "timezone": "Asia/Kolkata",
  "cron_expr": "0 8 * * *",
  "misfire_policy_key": "fire_once_now",
  "overlap_policy_key": "skip",
  "bridge": {
    "handler_key": "notify.digests.flush",
    "queue_key": "notify",
    "payload": { "mode": "DAILY" }
  },
  "priority": 100
}
```

Creates DRAFT version (or ACTIVE for system seeds per policy).

### 6.2 Versions

```http
GET  /api/v1/scheduler/schedules/{schedule_key}/versions
POST /api/v1/scheduler/schedules/{schedule_key}/versions
GET  /api/v1/scheduler/versions/{version_id}
PUT  /api/v1/scheduler/versions/{version_id}
```

Only DRAFT mutable.

### 6.3 Activate / pause / resume / retire

```http
POST /api/v1/scheduler/versions/{version_id}/activate
POST /api/v1/scheduler/schedules/{schedule_key}/pause
POST /api/v1/scheduler/schedules/{schedule_key}/resume
POST /api/v1/scheduler/schedules/{schedule_key}/retire
```

Activate computes `next_fire_at`. Pause stops future claims; inflight continue.

---

## 7. Simulate next fires

```http
POST /api/v1/scheduler/simulate
POST /api/v1/scheduler/versions/{version_id}/simulate
```

```json
{
  "from": "2026-09-09T00:00:00Z",
  "to": "2026-09-16T00:00:00Z",
  "limit": 50,
  "apply_calendar": true,
  "apply_blackout": true
}
```

**Response:** list of planned UTC instants + local wall times + skip annotations. No side effects (`SCH_SIMULATION_ONLY` if misused as run).

---

## 8. Manual run (primary ops)

```http
POST /api/v1/scheduler/schedules/{schedule_key}/run-now
Idempotency-Key: …
```

```json
{
  "payload_override": { "force": true },
  "ignore_blackout": false
}
```

Creates PLANNED/CLAIMED run with `manual_trigger` + enqueues immediately (subject to overlap policy). Requires `scheduler.run`. Critical ignore_blackout may need admin.

---

## 9. Runs ledger

```http
GET /api/v1/scheduler/runs?schedule_key=…&status=FAILED&from=…&to=…
GET /api/v1/scheduler/runs/{run_id}
GET /api/v1/scheduler/schedules/{schedule_key}/runs
POST /api/v1/scheduler/runs/{run_id}/cancel
```

Cancel only if PLANNED/CLAIMED/ENQUEUED (not RUNNING without cooperative cancel).

---

## 10. Calendars & blackouts

```http
GET  /api/v1/scheduler/calendars
POST /api/v1/scheduler/calendars
PUT  /api/v1/scheduler/calendars/{calendar_key}/days
GET  /api/v1/scheduler/blackouts
POST /api/v1/scheduler/blackouts
DELETE /api/v1/scheduler/blackouts/{id}
```

**Blackout:**

```json
{
  "start_at": "2026-03-28T00:00:00+05:30",
  "end_at": "2026-03-31T23:59:59+05:30",
  "reason": "FY close freeze",
  "tenant_id": null,
  "applies_to_critical": false
}
```

---

## 11. Dependencies

```http
GET /api/v1/scheduler/schedules/{schedule_key}/dependencies
PUT /api/v1/scheduler/schedules/{schedule_key}/dependencies
```

```json
{
  "upstream": [
    { "schedule_key": "etl.freight.daily", "condition": "SUCCESS", "max_wait_sec": 7200 }
  ]
}
```

Cycle detection → `SCH_DEPENDENCY_CYCLE`.

---

## 12. Policies & allowlists

```http
GET /api/v1/scheduler/policies/misfire
GET /api/v1/scheduler/policies/overlap
GET /api/v1/scheduler/handlers/allowlist
PUT /api/v1/scheduler/handlers/allowlist
```

---

## 13. Quotas

```http
GET /api/v1/scheduler/quotas
PUT /api/v1/scheduler/quotas/tenants/{tenant_id}
GET /api/v1/scheduler/quotas/tenants/{tenant_id}/usage
```

---

## 14. Tick engine (internal)

```http
POST /internal/v1/scheduler/tick
POST /internal/v1/scheduler/tick/dry-run
POST /internal/v1/scheduler/locks/heartbeat
POST /internal/v1/scheduler/shards/rebalance
```

**Tick:**

```json
{
  "ticker_key": "scheduler-1",
  "shard_no": 3,
  "limit": 100,
  "now": null
}
```

Claims due planned/next fires → applies blackout/overlap/misfire → enqueues p14 → writes runs → updates `next_fire_at`.

Dry-run returns would-be actions without enqueue.

---

## 15. Messaging callbacks (internal)

```http
POST /internal/v1/scheduler/runs/{run_id}/job-started
POST /internal/v1/scheduler/runs/{run_id}/job-succeeded
POST /internal/v1/scheduler/runs/{run_id}/job-failed
```

```json
{
  "messaging_job_id": "…",
  "error_code": "HANDLER_ERROR",
  "error_detail": "…"
}
```

Called by p14 bridge/middleware when schedule-origin jobs complete.

---

## 16. Packages & governance

```http
GET  /api/v1/scheduler/packages
POST /api/v1/scheduler/packages/{package_key}/install
GET  /api/v1/scheduler/changesets
POST /api/v1/scheduler/changesets
POST /api/v1/scheduler/changesets/{id}/approvals
```

---

## 17. Audit & metrics

```http
GET /api/v1/scheduler/stats?schedule_key=…&from=…&to=…
GET /api/v1/scheduler/tickers
GET /api/v1/scheduler/health
GET /api/v1/scheduler/audit?schedule_key=…
```

Health: ticker freshness, due backlog, failed run rate, lock contention.

---

## 18. Caching & concurrency

| Resource | Strategy |
|---|---|
| Active schedule defs | Cached; invalidate on activate |
| Due claim | SKIP LOCKED + shard lock |
| Planned fire unique | DB unique constraint |
| Manual run | Idempotency-Key |

---

## 19. Example flows

### 19.1 Daily digest

1. Cron `0 8 * * *` Asia/Kolkata ACTIVE  
2. Tick enqueues `notify.digests.flush`  
3. Callback marks SUCCEEDED  

### 19.2 Outage misfire

1. Ticker down 15 minutes  
2. On recovery, CATCH_UP_N=3 creates up to 3 catchup planned fires  
3. Then resumes normal next_fire  

### 19.3 Month-end blackout

1. Admin creates blackout window  
2. Non-critical report schedules SKIP with reason BLACKOUT  
3. System lease sweep (`ignore_blackout`) still fires  

### 19.4 Dependent ETL → report

1. Report schedule depends on ETL SUCCESS  
2. Report planned fire waits / skips if ETL failed  

---

## 20. Event hooks

| Event | Consumer |
|---|---|
| `scheduler.run.failed` | Notify ops |
| `scheduler.misfire.detected` | Monitoring |
| `scheduler.ticker.stale` | Autoscale/page |
| `scheduler.schedule.paused` | Audit |

---

## 21. Compatibility notes

- Public prefix `/api/v1/scheduler`; schema `scheduler`.  
- Cron dialect documented in guide (standard 5/6 field).  
- p10 process deadlines may call internal schedule one-shots or keep timers local — prefer p10 for instance-scoped timers; p17 for fleet-wide recurrence.  
- Always register handler_keys in allowlist before activate.

---

## 22. Related documents

- Guide: [`SCHEDULER_GUIDE.md`](SCHEDULER_GUIDE.md)  
- Schema: [`SCHEDULER_SCHEMA.md`](SCHEDULER_SCHEMA.md)  
- Messaging: [`../14_messaging/MESSAGING_API.md`](../14_messaging/MESSAGING_API.md)  
- Notification: [`../15_notification/NOTIFICATION_API.md`](../15_notification/NOTIFICATION_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
