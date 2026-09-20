# JeslotERP Scheduler Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — calendar + ticker/lock HTTP persist on AsyncSession  
**Package:** `platforms.p17_scheduler`  
**PostgreSQL schema:** `scheduler`  
**Companion:** [`SCHEDULER_GUIDE.md`](SCHEDULER_GUIDE.md) · [`SCHEDULER_API.md`](SCHEDULER_API.md)

> Runtime models: `platforms/p17_scheduler/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `scheduler` (never `p17`) |
| Tables | `sch_*` |
| Timestamps | Store UTC; interpret with `timezone` |
| Soft delete | Retire schedules; retain runs |
| Cross-schema | UUID refs (org fiscal calendar) |
| RLS | FORCE on tenant schedules/runs |
| Fire identity | `(schedule_id, planned_fire_at)` unique |

---

## 2. Complete table inventory (**58 tables**)

### 2.1 Catalog & definitions (10)

| # | Table | Purpose |
|---|---|---|
| 1 | `sch_schedule` | Schedule catalog |
| 2 | `sch_schedule_version` | Versioned definition |
| 3 | `sch_schedule_kind` | Kind catalog |
| 4 | `sch_cron_spec` | Cron expression details |
| 5 | `sch_interval_spec` | Interval specs |
| 6 | `sch_oneshot_spec` | One-shot fire_at |
| 7 | `sch_window_spec` | Windowed fire |
| 8 | `sch_activation` | Active version pointer |
| 9 | `sch_owner` | Ownership |
| 10 | `sch_feature_binding` | Feature gates |

### 2.2 Policies (7)

| # | Table | Purpose |
|---|---|---|
| 11 | `sch_misfire_policy` | Misfire behaviors |
| 12 | `sch_overlap_policy` | Overlap behaviors |
| 13 | `sch_jitter_policy` | Jitter |
| 14 | `sch_catchup_policy` | Catch-up bounds |
| 15 | `sch_priority_policy` | Priority bands |
| 16 | `sch_retry_hint` | Hint for p14 enqueue |
| 17 | `sch_policy_bind` | Bind policies → schedule version |

### 2.3 Calendars & blackouts (7)

| # | Table | Purpose |
|---|---|---|
| 18 | `sch_calendar` | Holiday/business calendars |
| 19 | `sch_calendar_day` | Explicit days |
| 20 | `sch_calendar_rule` | Recurring exclude/include rules |
| 21 | `sch_blackout_window` | Blackout periods |
| 22 | `sch_blackout_scope` | Global/tenant/schedule scope |
| 23 | `sch_fiscal_trigger` | FY/period trigger refs |
| 24 | `sch_timezone` | Allowed TZ catalog subset |

### 2.4 Dependencies (4)

| # | Table | Purpose |
|---|---|---|
| 25 | `sch_dependency` | Upstream/downstream |
| 26 | `sch_dependency_condition` | SUCCESS/COMPLETE/etc. |
| 27 | `sch_dependency_wait` | Waiting runs |
| 28 | `sch_dag_snapshot` | Optional DAG meta |

### 2.5 Runtime fires & runs (10)

| # | Table | Purpose |
|---|---|---|
| 29 | `sch_next_fire` | Materialized next fire |
| 30 | `sch_planned_fire` | Planned fire instances |
| 31 | `sch_run` | Run ledger |
| 32 | `sch_run_attempt` | Enqueue/callback attempts |
| 33 | `sch_run_skip` | Skip reasons |
| 34 | `sch_run_link` | Links to p14 job_id / event_id |
| 35 | `sch_manual_trigger` | Manual run requests |
| 36 | `sch_catchup_batch` | Catch-up batches |
| 37 | `sch_run_stats` | Aggregates |
| 38 | `sch_dead_schedule_alert` | Health alerts |

### 2.6 Ticker / locks / shards (7)

| # | Table | Purpose |
|---|---|---|
| 39 | `sch_ticker` | Ticker instances |
| 40 | `sch_tick_lock` | Distributed locks |
| 41 | `sch_tick_shard` | Sharding keys |
| 42 | `sch_tick_run` | Tick executions |
| 43 | `sch_tick_metric` | Tick latency/due counts |
| 44 | `sch_lease_heartbeat` | Ticker heartbeats |
| 45 | `sch_rebalance_event` | Shard rebalance |

### 2.7 Bridges & payloads (5)

| # | Table | Purpose |
|---|---|---|
| 46 | `sch_bridge_messaging` | → p14 mapping |
| 47 | `sch_payload_template` | Payload JSON templates |
| 48 | `sch_payload_param` | Declared params |
| 49 | `sch_bridge_event` | Optional p13 emit |
| 50 | `sch_handler_allowlist` | Allowed handler_keys |

### 2.8 Quotas, governance, packs (8)

| # | Table | Purpose |
|---|---|---|
| 51 | `sch_tenant_quota` | Max schedules / fires |
| 52 | `sch_tenant_usage` | Usage |
| 53 | `sch_changeset` | Definition changes |
| 54 | `sch_approval` | Approvals |
| 55 | `sch_package` | Packs |
| 56 | `sch_package_item` | Items |
| 57 | `sch_simulation_run` | Next-fire simulations |
| 58 | `sch_catalog_audit` | Audit |

**Plumbing:** `sch_outbox`, `sch_idempotency_key`

**Implementation total with plumbing: 60 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `sch_kind` | `CRON`, `INTERVAL`, `ONE_SHOT`, `WINDOW`, `CALENDAR_EVENT`, `DEPENDENT` |
| `sch_lifecycle` | `DRAFT`, `ACTIVE`, `PAUSED`, `RETIRED` |
| `sch_run_status` | `PLANNED`, `CLAIMED`, `ENQUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `SKIPPED`, `CANCELLED`, `MISFIRED` |
| `sch_misfire` | `FIRE_ONCE_NOW`, `SKIP`, `CATCH_UP_N`, `FIRE_ALL_BOUNDED` |
| `sch_overlap` | `SKIP`, `QUEUE_ONE`, `ALLOW_PARALLEL`, `FORBID` |
| `sch_skip_reason` | `OVERLAP`, `BLACKOUT`, `FEATURE_OFF`, `DEPENDENCY`, `PAUSED`, `QUOTA`, `MANUAL` |
| `sch_calendar_day_type` | `HOLIDAY`, `BUSINESS`, `FORCE_OFF`, `FORCE_ON` |

---

## 4. Definitions (detail)

### 4.1 `sch_schedule`

| Column | Type | Notes |
|---|---|---|
| `schedule_key` | VARCHAR(120) | Unique with tenant scope |
| `tenant_id` | UUID NULL | Null = system |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `lifecycle` | VARCHAR(20) | |
| `priority` | INT | Lower first |
| `owner_platform` | VARCHAR(80) NULL | |
| `ignore_blackout` | BOOLEAN DEFAULT false | |
| `is_system` | BOOLEAN | |

**Unique:** `(COALESCE(tenant_id, zero), schedule_key)`.

### 4.2 `sch_schedule_version`

| Column | Type | Notes |
|---|---|---|
| `schedule_id` | UUID | |
| `version_number` | INT | |
| `kind` | VARCHAR(20) | |
| `timezone` | VARCHAR(100) | IANA |
| `start_at` / `end_at` | TIMESTAMPTZ NULL | Validity |
| `checksum` | VARCHAR(64) | |
| `notes` | TEXT NULL | |

### 4.3 `sch_cron_spec`

| Column | Type | Notes |
|---|---|---|
| `version_id` | UUID | |
| `cron_expr` | VARCHAR(100) | |
| `include_seconds` | BOOLEAN | |
| `description` | VARCHAR(200) NULL | |

### 4.4 `sch_interval_spec`

| Column | Type | Notes |
|---|---|---|
| `version_id` | UUID | |
| `interval_seconds` | INT | |
| `anchor_at` | TIMESTAMPTZ NULL | |

### 4.5 `sch_activation`

| Column | Type | Notes |
|---|---|---|
| `schedule_id` | UUID UNIQUE | |
| `version_id` | UUID | |
| `activated_at` | TIMESTAMPTZ | |
| `next_fire_at` | TIMESTAMPTZ NULL | Denorm hot index |
| `last_fire_at` | TIMESTAMPTZ NULL | |

---

## 5. Policies

### 5.1 `sch_misfire_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | |
| `mode` | VARCHAR(30) | |
| `catchup_max` | INT NULL | For CATCH_UP_N |
| `misfire_threshold_sec` | INT | How late is a misfire |

### 5.2 `sch_overlap_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | |
| `mode` | VARCHAR(30) | |
| `queue_one_timeout_sec` | INT NULL | |

### 5.3 `sch_jitter_policy`

| Column | Type | Notes |
|---|---|---|
| `max_jitter_sec` | INT | Random 0..max added to fire |

---

## 6. Calendars

### 6.1 `sch_calendar`

| Column | Type | Notes |
|---|---|---|
| `calendar_key` | VARCHAR(100) | |
| `tenant_id` | UUID NULL | |
| `timezone` | VARCHAR(100) | |
| `name` | VARCHAR(150) | |

### 6.2 `sch_calendar_day`

| Column | Type | Notes |
|---|---|---|
| `calendar_id` | UUID | |
| `day` | DATE | |
| `day_type` | VARCHAR(20) | |
| `label` | VARCHAR(100) NULL | |

### 6.3 `sch_blackout_window`

| Column | Type | Notes |
|---|---|---|
| `start_at` / `end_at` | TIMESTAMPTZ | |
| `reason` | TEXT | |
| `applies_to_critical` | BOOLEAN DEFAULT false | |

### 6.4 `sch_fiscal_trigger`

| Column | Type | Notes |
|---|---|---|
| `version_id` | UUID | |
| `org_fiscal_calendar_id` | UUID | Org ref |
| `event_type` | VARCHAR(40) | `PERIOD_OPEN`, `FY_OPEN`, `PERIOD_CLOSE` |

---

## 7. Runs & fires

### 7.1 `sch_planned_fire`

| Column | Type | Notes |
|---|---|---|
| `schedule_id` | UUID | |
| `version_id` | UUID | |
| `planned_fire_at` | TIMESTAMPTZ | UTC instant |
| `status` | VARCHAR(20) | PLANNED/CLAIMED/… |
| `is_catchup` | BOOLEAN | |

**Unique:** `(schedule_id, planned_fire_at)`.

### 7.2 `sch_run`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | |
| `schedule_id` | UUID | |
| `planned_fire_id` | UUID | |
| `status` | VARCHAR(20) | |
| `claimed_by_ticker` | VARCHAR(100) NULL | |
| `enqueued_at` | TIMESTAMPTZ NULL | |
| `started_at` / `finished_at` | TIMESTAMPTZ NULL | |
| `skip_reason` | VARCHAR(30) NULL | |
| `error_code` | VARCHAR(50) NULL | |
| `correlation_id` | UUID NULL | |
| `manual_trigger_id` | UUID NULL | |

### 7.3 `sch_run_link`

| Column | Type | Notes |
|---|---|---|
| `run_id` | UUID | |
| `messaging_job_id` | UUID NULL | p14 |
| `event_id` | UUID NULL | p13 optional |

### 7.4 `sch_manual_trigger`

| Column | Type | Notes |
|---|---|---|
| `schedule_id` | UUID | |
| `requested_by` | UUID | |
| `requested_at` | TIMESTAMPTZ | |
| `payload_override` | JSONB NULL | |
| `run_id` | UUID NULL | |

---

## 8. Ticker locks

### 8.1 `sch_tick_lock`

| Column | Type | Notes |
|---|---|---|
| `lock_key` | VARCHAR(100) UNIQUE | e.g. `shard:3` |
| `owner_ticker` | VARCHAR(100) | |
| `expires_at` | TIMESTAMPTZ | |
| `heartbeat_at` | TIMESTAMPTZ | |

### 8.2 `sch_tick_shard`

| Column | Type | Notes |
|---|---|---|
| `shard_no` | INT | |
| `key_range` | VARCHAR(50) | hash ranges of schedule_id |
| `is_active` | BOOLEAN | |

---

## 9. Bridges

### 9.1 `sch_bridge_messaging`

| Column | Type | Notes |
|---|---|---|
| `version_id` | UUID | |
| `handler_key` | VARCHAR(150) | Must be allow-listed |
| `queue_key` | VARCHAR(100) | |
| `payload_template_id` | UUID | |
| `priority` | VARCHAR(20) NULL | p14 priority |

Payload template may include `{{schedule_key}}`, `{{planned_fire_at}}`, `{{tenant_id}}`, static JSON.

---

## 10. Dependencies

### 10.1 `sch_dependency`

| Column | Type | Notes |
|---|---|---|
| `schedule_id` | UUID | Downstream |
| `upstream_schedule_id` | UUID | |
| `condition` | VARCHAR(20) | SUCCESS |
| `max_wait_sec` | INT NULL | |

Downstream planned fire waits until upstream SUCCEEDED for aligned period (policy).

---

## 11. Governance & packs

Seed packs:

- `scheduler.system.messaging_sweeps@1.0.0` — lease/delayed sweeps  
- `scheduler.system.event_bus_ticks@1.0.0`  
- `scheduler.system.notify_digest@1.0.0`  
- `scheduler.system.cache_warmup@1.0.0`  

Approvals for production cron changes on critical keys.

---

## 12. Plumbing

| Table | Purpose |
|---|---|
| `sch_outbox` | Domain events |
| `sch_idempotency_key` | Manual trigger / activate |

---

## 13. RLS summary

| Class | Policy |
|---|---|
| System schedules | Read auth; manage permission |
| Tenant schedules/runs | FORCE `tenant_id` |
| Global blackouts | Admin |
| Tick locks | Service identity |

---

## 14. Seed minimum

1. Timezones allowlist including `Asia/Kolkata`, `UTC`  
2. Misfire/overlap default policies  
3. System schedules for messaging sweeps & event relay ticks  
4. Handler allowlist matching p14 seeds  
5. Permissions `scheduler.*`  
6. Empty holiday calendar template  

---

## 15. ER overview

```text
schedule ── versions ── cron/interval/oneshot/window specs
                     ── policy binds
                     ── messaging bridge / payload template
                     ── calendars / fiscal triggers
                     ── dependencies

activation ── next_fire_at
planned_fire ── run ── links (p14/p13)
ticker ── locks / shards / tick_runs
blackout_windows
packages
```

---

## 16. Implementation notes

1. Due scan: `next_fire_at <= now()` AND lifecycle ACTIVE AND not paused — `FOR UPDATE SKIP LOCKED` by shard.  
2. After enqueue success, compute and write next `next_fire_at`.  
3. p14 callback updates run RUNNING→SUCCEEDED/FAILED.  
4. Never delete run history; retain per policy.  
5. Split models: `catalog`, `policy`, `calendar`, `runtime`, `ticker`, `bridge`, `governance`, `plumbing`.
