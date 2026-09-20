# JeslotERP Messaging Platform — Production Schema (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `msg_job` / `msg_job_payload` HTTP persist on AsyncSession. Broker is a port; Redis/Rabbit are `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p14_messaging`  
**PostgreSQL schema:** `messaging`  
**Companion:** [`MESSAGING_GUIDE.md`](MESSAGING_GUIDE.md) · [`MESSAGING_API.md`](MESSAGING_API.md) · [`MESSAGING_RTM.md`](MESSAGING_RTM.md)

> Runtime models: `platforms/p14_messaging/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `messaging` (never `p14`) |
| Tables | `msg_*` |
| Handler keys | Dot namespaces (`media.scan_file`) |
| Soft delete | Cancel/archive jobs; retain DLQ history |
| Cross-schema | UUID refs in payload only |
| RLS | FORCE when `tenant_id` present on jobs |
| Hot path | Indexed `(queue_id, status, run_at, priority)` |

---

## 2. Complete table inventory (**60 tables**)

### 2.1 Queues & policies (10)

| # | Table | Purpose |
|---|---|---|
| 1 | `msg_queue` | Queue catalog |
| 2 | `msg_queue_kind` | Kind catalog |
| 3 | `msg_queue_policy` | Retry/rate/concurrency |
| 4 | `msg_priority_lane` | Priority lanes |
| 5 | `msg_retry_policy` | Retry definitions |
| 6 | `msg_backoff_policy` | Backoff curves |
| 7 | `msg_rate_limit_policy` | Rate limits |
| 8 | `msg_fairness_policy` | Tenant fairness |
| 9 | `msg_depth_policy` | Soft/hard depth |
| 10 | `msg_queue_state` | Paused/draining flags |

### 2.2 Handlers & workers (8)

| # | Table | Purpose |
|---|---|---|
| 11 | `msg_handler` | Allow-listed handlers |
| 12 | `msg_handler_param_schema` | Payload JSON schema |
| 13 | `msg_handler_queue_bind` | Default queue binds |
| 14 | `msg_worker` | Worker instances |
| 15 | `msg_worker_capability` | Handler capabilities |
| 16 | `msg_worker_heartbeat` | Heartbeats |
| 17 | `msg_worker_session` | Session metrics |
| 18 | `msg_worker_version` | Deployed versions |

### 2.3 Jobs / messages (10)

| # | Table | Purpose |
|---|---|---|
| 19 | `msg_job` | Job header |
| 20 | `msg_job_payload` | Payload body |
| 21 | `msg_job_header` | Indexed headers |
| 22 | `msg_job_attempt` | Attempt history |
| 23 | `msg_job_error` | Error details |
| 24 | `msg_job_result` | Small result digest |
| 25 | `msg_job_tag` | Tags |
| 26 | `msg_job_link` | Correlation to entities |
| 27 | `msg_job_dependency` | Job waits on job |
| 28 | `msg_idempotency` | Enqueue idempotency |

### 2.4 Leases & execution (6)

| # | Table | Purpose |
|---|---|---|
| 29 | `msg_lease` | Active leases |
| 30 | `msg_lease_history` | Lease history |
| 31 | `msg_visibility` | Visibility timeouts |
| 32 | `msg_heartbeat_event` | Lease extensions |
| 33 | `msg_cancellation` | Cancel requests |
| 34 | `msg_poison_record` | Poison quarantine |

### 2.5 FIFO / delay / priority (5)

| # | Table | Purpose |
|---|---|---|
| 35 | `msg_fifo_group` | FIFO group state |
| 36 | `msg_fifo_seq` | Sequence per group |
| 37 | `msg_delay_index` | Due index helper |
| 38 | `msg_priority_score` | Computed score cache |
| 39 | `msg_ready_pointer` | Optional ready-queue pointer |

### 2.6 DLQ & redrive (5)

| # | Table | Purpose |
|---|---|---|
| 40 | `msg_dlq_entry` | Dead letters |
| 41 | `msg_dlq_reason` | Reasons |
| 42 | `msg_redrive_job` | Redrive batches |
| 43 | `msg_redrive_item` | Items |
| 44 | `msg_discard_log` | Discard audit |

### 2.7 Batch jobs (5)

| # | Table | Purpose |
|---|---|---|
| 45 | `msg_batch` | Batch parent |
| 46 | `msg_batch_item` | Child job links |
| 47 | `msg_batch_policy` | Fan-out policies |
| 48 | `msg_batch_progress` | Progress counters |
| 49 | `msg_chunk_spec` | Chunking specs |

### 2.8 Bridges, fairness runtime, governance (11)

| # | Table | Purpose |
|---|---|---|
| 50 | `msg_bridge_subscription` | p13 → job bridge |
| 51 | `msg_bridge_scheduler` | p17 → job templates |
| 52 | `msg_tenant_quota` | Per-tenant quotas |
| 53 | `msg_tenant_usage` | Runtime usage |
| 54 | `msg_queue_metrics` | Depth/lag samples |
| 55 | `msg_changeset` | Queue policy changes |
| 56 | `msg_approval` | Approvals |
| 57 | `msg_package` | Queue/handler packs |
| 58 | `msg_package_item` | Pack items |
| 59 | `msg_admin_audit` | Pause/purge audit |
| 60 | `msg_feature_binding` | Feature gates |

**Plumbing:** `msg_outbox`, `msg_idempotency_key` (API), `msg_catalog_audit`

**Implementation total with plumbing: 63 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `msg_queue_kind` | `STANDARD`, `FIFO`, `DELAY`, `PRIORITY`, `BATCH` |
| `msg_job_status` | `PENDING`, `DELAYED`, `READY`, `LEASED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `DEAD_LETTER`, `CANCELLED`, `POISON` |
| `msg_priority` | `CRITICAL`, `HIGH`, `NORMAL`, `LOW` |
| `msg_lease_status` | `ACTIVE`, `EXPIRED`, `RELEASED`, `STOLEN` |
| `msg_dlq_reason` | `MAX_ATTEMPTS`, `TIMEOUT`, `HANDLER_ERROR`, `CANCELLED`, `POISON`, `REJECTED` |
| `msg_batch_status` | `OPEN`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED` |
| `msg_queue_run_state` | `RUNNING`, `PAUSED`, `DRAINING` |
| `msg_backoff_kind` | `FIXED`, `EXPONENTIAL`, `EXPO_JITTER` |

---

## 4. Queues (detail)

### 4.1 `msg_queue`

| Column | Type | Notes |
|---|---|---|
| `queue_key` | VARCHAR(100) UNIQUE | `default`, `media`, `notify`, `search` |
| `name` | VARCHAR(150) | |
| `kind` | VARCHAR(20) | |
| `is_system` | BOOLEAN | |
| `is_active` | BOOLEAN | |
| `policy_id` | UUID | |
| `default_timeout_ms` | INT | |
| `default_max_attempts` | INT | |
| `visibility_timeout_sec` | INT | |

### 4.2 `msg_queue_policy`

| Column | Type | Notes |
|---|---|---|
| `max_concurrency` | INT | |
| `max_depth` | BIGINT | Hard |
| `soft_depth` | BIGINT | Shed LOW |
| `retry_policy_id` | UUID | |
| `rate_limit_policy_id` | UUID NULL | |
| `fairness_policy_id` | UUID NULL | |
| `allow_fifo` | BOOLEAN | |

### 4.3 `msg_queue_state`

| Column | Type | Notes |
|---|---|---|
| `queue_id` | UUID UNIQUE | |
| `run_state` | VARCHAR(20) | |
| `paused_by` | UUID NULL | |
| `paused_at` | TIMESTAMPTZ NULL | |
| `pause_reason` | TEXT NULL | |

---

## 5. Handlers & workers

### 5.1 `msg_handler`

| Column | Type | Notes |
|---|---|---|
| `handler_key` | VARCHAR(150) UNIQUE | |
| `description` | TEXT NULL | |
| `default_queue_key` | VARCHAR(100) | |
| `timeout_ms` | INT | |
| `max_attempts` | INT NULL | Override |
| `is_active` | BOOLEAN | |
| `requires_tenant` | BOOLEAN | |
| `payload_schema_id` | UUID NULL | |
| `owner_platform` | VARCHAR(80) | |

### 5.2 `msg_worker`

| Column | Type | Notes |
|---|---|---|
| `worker_key` | VARCHAR(100) | Instance id |
| `hostname` | VARCHAR(150) NULL | |
| `version` | VARCHAR(50) NULL | |
| `status` | VARCHAR(20) | ONLINE/STALE/OFFLINE |
| `last_heartbeat_at` | TIMESTAMPTZ | |
| `max_leases` | INT | |

### 5.3 `msg_worker_capability`

Lists `handler_key` or `queue_key` patterns the worker accepts.

---

## 6. Jobs

HTTP enqueue persists `msg_job` + `msg_job_payload` on `AsyncSession`. Empty job list is `[]`. Claim/succeed/fail update `job_status` when a row exists. The in-memory `MessagingCatalogStore` is the TestClient double. Redis/Rabbit are a `BrokerPort` — ping/test-connection are `PROVIDER_PENDING` until a real broker is configured. No invented queue hits.

### 6.1 `msg_job`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | RLS when set |
| `company_id` | UUID NULL | |
| `queue_id` | UUID | |
| `handler_key` | VARCHAR(150) | |
| `status` | VARCHAR(20) | |
| `priority` | VARCHAR(20) | |
| `priority_score` | INT | Lower = sooner |
| `run_at` | TIMESTAMPTZ | |
| `group_key` | VARCHAR(200) NULL | FIFO |
| `group_seq` | BIGINT NULL | |
| `idempotency_key` | VARCHAR(120) NULL | |
| `correlation_id` | UUID NULL | |
| `causation_id` | UUID NULL | |
| `attempt_count` | INT | |
| `max_attempts` | INT | |
| `timeout_ms` | INT | |
| `batch_id` | UUID NULL | |
| `parent_job_id` | UUID NULL | |
| `scheduled_by` | VARCHAR(40) NULL | api/scheduler/event_bus |
| `created_at` | TIMESTAMPTZ | |
| `started_at` / `finished_at` | TIMESTAMPTZ NULL | |

**Partial unique:** `(queue_id, idempotency_key)` where key not null and status not terminal-discarded.

### 6.2 `msg_job_payload`

| Column | Type | Notes |
|---|---|---|
| `job_id` | UUID UNIQUE | |
| `payload` | JSONB | |
| `is_sensitive` | BOOLEAN | |
| `checksum` | VARCHAR(64) | |

### 6.3 `msg_idempotency`

| Column | Type | Notes |
|---|---|---|
| `queue_id` | UUID | |
| `idempotency_key` | VARCHAR(120) | |
| `job_id` | UUID | |
| `expires_at` | TIMESTAMPTZ | |

---

## 7. Leases

### 7.1 `msg_lease`

| Column | Type | Notes |
|---|---|---|
| `job_id` | UUID UNIQUE active | |
| `worker_id` | UUID | |
| `status` | VARCHAR(20) | |
| `leased_at` | TIMESTAMPTZ | |
| `lease_expires_at` | TIMESTAMPTZ | |
| `heartbeat_at` | TIMESTAMPTZ | |
| `attempt_no` | INT | |

Expired lease sweeper sets job READY/FAILED per policy and closes lease EXPIRED.

---

## 8. FIFO

### 8.1 `msg_fifo_group`

| Column | Type | Notes |
|---|---|---|
| `queue_id` | UUID | |
| `group_key` | VARCHAR(200) | |
| `next_seq` | BIGINT | |
| `inflight_job_id` | UUID NULL | Current leased/running |
| `is_blocked` | BOOLEAN | On poison |

**Unique:** `(queue_id, group_key)`.

Only one inflight job per group; next READY with `group_seq = last_acked+1`.

---

## 9. DLQ

### 9.1 `msg_dlq_entry`

| Column | Type | Notes |
|---|---|---|
| `job_id` | UUID | |
| `queue_id` | UUID | |
| `reason` | VARCHAR(30) | |
| `last_error_code` | VARCHAR(50) NULL | |
| `enqueued_at` | TIMESTAMPTZ | |
| `status` | VARCHAR(20) | OPEN/REDRIVEN/DISCARDED |

---

## 10. Batch

### 10.1 `msg_batch`

| Column | Type | Notes |
|---|---|---|
| `batch_key` | VARCHAR(120) | |
| `handler_key` | VARCHAR(150) | Coordinator |
| `status` | VARCHAR(20) | |
| `total_items` | INT | |
| `succeeded_items` | INT | |
| `failed_items` | INT | |
| `tenant_id` | UUID NULL | |

### 10.2 `msg_batch_item`

Links `batch_id` → `job_id` with `chunk_index`.

---

## 11. Bridges & quotas

### 11.1 `msg_bridge_subscription`

| Column | Type | Notes |
|---|---|---|
| `event_subscription_key` | VARCHAR(150) | p13 |
| `handler_key` | VARCHAR(150) | |
| `queue_key` | VARCHAR(100) | |
| `payload_map` | JSONB | CE data → job payload |
| `is_active` | BOOLEAN | |

### 11.2 `msg_tenant_quota`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `queue_id` | UUID NULL | Null = global |
| `max_enqueue_per_min` | INT | |
| `max_inflight` | INT | |

---

## 12. Governance & packs

- Packages seed queues: `default`, `critical`, `media`, `notify`, `search`, `event_dispatch`, `reports`  
- Handlers metadata for system platforms  
- Approvals for production queue policy tightening  

---

## 13. Plumbing

| Table | Purpose |
|---|---|
| `msg_outbox` | Messaging domain events |
| `msg_idempotency_key` | Admin API idempotency |
| `msg_catalog_audit` | Queue/handler audit |

Also: `msg_feature_binding` for feature-gated queues.

---

## 14. RLS summary

| Class | Policy |
|---|---|
| Queues/handlers system | Read auth; manage permission |
| Jobs with tenant_id | FORCE RLS |
| DLQ/batches tenant | FORCE when tenant set |
| Workers | Service identity; not tenant |

---

## 15. Seed minimum

1. Queues listed above  
2. Retry `default_exponential` (5 attempts, jitter)  
3. Fairness default weights  
4. Handlers placeholders: `messaging.noop`, `notify.dispatch_channel`, `media.scan_file`, `search.reindex_chunk`, `event_bus.dispatch_push`  
5. Permissions `messaging.*`  
6. Depth soft/hard defaults  

---

## 16. ER overview

```text
queue ── policies / state
handler ── binds → queue
worker ── capabilities / heartbeats
job ── payload / attempts / lease
   ├── fifo_group
   ├── dlq_entry
   └── batch_item ← batch

bridges (event_bus / scheduler)
tenant_quota / metrics
packages
```

---

## 17. Implementation notes

1. Claim query: `FOR UPDATE SKIP LOCKED` on READY due jobs respecting pause/fairness/FIFO.  
2. Heartbeat must extend `lease_expires_at` atomically.  
3. Sensitive payloads redacted in admin list APIs.  
4. Optional Redis broker is an accelerator — **DB ledger remains source of truth**.  
5. Split models: `queue`, `handler`, `job`, `lease`, `fifo`, `dlq`, `batch`, `bridge`, `governance`, `plumbing`.
