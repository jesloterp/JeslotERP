# JeslotERP Monitoring Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `mon_metric` / `mon_scrape_target` are the HTTP catalog ledger. Samples stay on TSDB port. Not Production.  
**Package:** `platforms.p21_monitoring`  
**PostgreSQL schema:** `monitoring`  
**Companion:** [`MONITORING_GUIDE.md`](MONITORING_GUIDE.md) · [`MONITORING_API.md`](MONITORING_API.md)

> Runtime models: `platforms/p21_monitoring/infrastructure/persistence/models/`.  
> **Note:** High-volume samples/spans live in TSDB/trace backends; Postgres holds catalog, SLO, alerts, probes, dashboards.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `monitoring` (never `p21`) |
| Tables | `mon_*` |
| Soft delete | Retire rules; keep alert history |
| Cross-schema | UUID refs only |
| RLS | Tenant business metrics FORCE; infra global |
| Secrets | Pager/webhook via configuration `secret_ref` |
| Cardinality | Enforced on label keys |

---

## 2. Complete table inventory (**60 tables**)

### 2.1 Services & topology (7)

| # | Table | Purpose |
|---|---|---|
| 1 | `mon_service` | Monitored services |
| 2 | `mon_component` | Components within service |
| 3 | `mon_dependency` | Declared deps |
| 4 | `mon_environment` | envs |
| 5 | `mon_team` | Owning teams |
| 6 | `mon_service_team` | M2M |
| 7 | `mon_feature_binding` | Feature gates |

### 2.2 Metrics catalog (8)

| # | Table | Purpose |
|---|---|---|
| 8 | `mon_metric` | Metric defs |
| 9 | `mon_metric_label` | Allowed labels |
| 10 | `mon_metric_unit` | Units |
| 11 | `mon_recording_rule` | Recording rules |
| 12 | `mon_metric_backend` | TSDB backends |
| 13 | `mon_scrape_target` | Scrape targets |
| 14 | `mon_exemplar_policy` | Exemplars to traces |
| 15 | `mon_cardinality_guard` | Guards |

### 2.3 SLI / SLO (7)

| # | Table | Purpose |
|---|---|---|
| 16 | `mon_sli` | SLI definitions |
| 17 | `mon_slo` | SLO objectives |
| 18 | `mon_slo_window` | Rolling windows |
| 19 | `mon_error_budget` | Budget state snapshots |
| 20 | `mon_burn_rate_policy` | Burn alert policies |
| 21 | `mon_slo_report` | Reports |
| 22 | `mon_journey` | Critical user journeys |

### 2.4 Health probes (6)

| # | Table | Purpose |
|---|---|---|
| 23 | `mon_probe` | Probe defs |
| 24 | `mon_probe_step` | Multi-step probes |
| 25 | `mon_probe_result` | Recent results |
| 26 | `mon_probe_status` | Current status |
| 27 | `mon_dependency_health` | Aggregated dep health |
| 28 | `mon_status_page` | Public/internal status |

### 2.5 Traces (5)

| # | Table | Purpose |
|---|---|---|
| 29 | `mon_trace_backend` | Tempo/Jaeger/OTel |
| 30 | `mon_trace_policy` | Sampling policies |
| 31 | `mon_slow_span_rule` | Slow span detection |
| 32 | `mon_trace_tail_sample` | Tail sample rules |
| 33 | `mon_service_map_edge` | Derived/cached edges |

### 2.6 Alerts & routing (12)

| # | Table | Purpose |
|---|---|---|
| 34 | `mon_alert_rule` | Alert rules |
| 35 | `mon_alert_condition` | Conditions |
| 36 | `mon_alert_event` | Firing instances |
| 37 | `mon_alert_history` | State transitions |
| 38 | `mon_severity` | INFO/WARN/CRIT |
| 39 | `mon_route` | Notification routes |
| 40 | `mon_route_match` | Matchers |
| 41 | `mon_receiver` | Receivers (notify/webhook/pager) |
| 42 | `mon_escalation_policy` | Escalations |
| 43 | `mon_escalation_step` | Steps |
| 44 | `mon_silence` | Silences |
| 45 | `mon_inhibit_rule` | Inhibition |

### 2.7 Dashboards, synthetics, incidents (8)

| # | Table | Purpose |
|---|---|---|
| 46 | `mon_dashboard` | Dashboard defs |
| 47 | `mon_dashboard_version` | Versions |
| 48 | `mon_panel` | Panels meta |
| 49 | `mon_synthetic_check` | Synthetics |
| 50 | `mon_synthetic_run` | Run results |
| 51 | `mon_synthetic_location` | Probe locations |
| 52 | `mon_incident` | Lite incidents |
| 53 | `mon_incident_alert` | Alerts linked |

### 2.8 Capacity, governance, packs (7)

| # | Table | Purpose |
|---|---|---|
| 54 | `mon_capacity_signal` | Saturation signals |
| 55 | `mon_runbook` | Runbook links |
| 56 | `mon_changeset` | Rule changes |
| 57 | `mon_approval` | Approvals |
| 58 | `mon_package` | Packs |
| 59 | `mon_package_item` | Items |
| 60 | `mon_catalog_audit` | Audit |

**Plumbing:** `mon_outbox`, `mon_idempotency_key`

**Implementation total with plumbing: 62 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `mon_metric_type` | `COUNTER`, `GAUGE`, `HISTOGRAM`, `SUMMARY` |
| `mon_probe_kind` | `LIVE`, `READY`, `DEPENDENCY`, `SYNTHETIC` |
| `mon_probe_state` | `HEALTHY`, `DEGRADED`, `DOWN`, `UNKNOWN` |
| `mon_alert_state` | `PENDING`, `FIRING`, `ACKED`, `RESOLVED`, `SILENCED` |
| `mon_severity` | `INFO`, `WARNING`, `CRITICAL` |
| `mon_slo_compliance` | `HEALTHY`, `BREACHING`, `EXHAUSTED` |
| `mon_receiver_kind` | `NOTIFY_TOPIC`, `WEBHOOK`, `PAGER`, `EMAIL_ALIAS` |

---

## 4. Services & metrics

### 4.1 `mon_service`

| Column | Type | Notes |
|---|---|---|
| `service_key` | VARCHAR(80) UNIQUE | matches logging service |
| `name` | VARCHAR(150) | |
| `tier` | VARCHAR(20) | CRITICAL/HIGH/NORMAL |
| `owner_team_id` | UUID NULL | |
| `repo` | VARCHAR(200) NULL | |

### 4.2 `mon_metric`

| Column | Type | Notes |
|---|---|---|
| `metric_key` | VARCHAR(150) UNIQUE | `http_server_requests_seconds` |
| `metric_type` | VARCHAR(20) | |
| `unit` | VARCHAR(30) | |
| `description` | TEXT | |
| `owner_service` | VARCHAR(80) NULL | |
| `is_active` | BOOLEAN | |

### 4.3 `mon_metric_label`

| Column | Type | Notes |
|---|---|---|
| `metric_id` | UUID | |
| `label_key` | VARCHAR(50) | |
| `allowed_values` | JSONB NULL | Optional enum |
| `max_cardinality_hint` | INT NULL | |

---

## 5. SLO

### 5.1 `mon_sli`

| Column | Type | Notes |
|---|---|---|
| `sli_key` | VARCHAR(100) UNIQUE | |
| `service_id` | UUID | |
| `good_query` | TEXT | PromQL/OTel |
| `total_query` | TEXT | |
| `description` | TEXT NULL | |

### 5.2 `mon_slo`

| Column | Type | Notes |
|---|---|---|
| `slo_key` | VARCHAR(100) UNIQUE | |
| `sli_id` | UUID | |
| `objective` | NUMERIC(6,5) | 0.999 |
| `window_days` | INT | 28 |
| `journey_id` | UUID NULL | |

### 5.3 `mon_burn_rate_policy`

| Column | Type | Notes |
|---|---|---|
| `slo_id` | UUID | |
| `short_window` | VARCHAR(20) | `1h` |
| `long_window` | VARCHAR(20) | `6h` |
| `burn_threshold` | NUMERIC | e.g. 14.4 |
| `severity` | VARCHAR(20) | |

---

## 6. Probes

### 6.1 `mon_probe`

| Column | Type | Notes |
|---|---|---|
| `probe_key` | VARCHAR(100) UNIQUE | |
| `kind` | VARCHAR(20) | |
| `service_id` | UUID NULL | |
| `target_url` | TEXT NULL | |
| `interval_sec` | INT | |
| `timeout_ms` | INT | |
| `expected_status` | INT NULL | |
| `is_active` | BOOLEAN | |

### 6.2 `mon_probe_status`

| Column | Type | Notes |
|---|---|---|
| `probe_id` | UUID UNIQUE | |
| `state` | VARCHAR(20) | |
| `last_checked_at` | TIMESTAMPTZ | |
| `last_success_at` | TIMESTAMPTZ NULL | |
| `message` | TEXT NULL | |

---

## 7. Alerts

### 7.1 `mon_alert_rule`

| Column | Type | Notes |
|---|---|---|
| `rule_key` | VARCHAR(120) UNIQUE | |
| `severity` | VARCHAR(20) | |
| `query` | TEXT | |
| `for_duration` | VARCHAR(20) | `5m` |
| `summary` | VARCHAR(200) | |
| `description` | TEXT NULL | |
| `runbook_id` | UUID NULL | |
| `is_active` | BOOLEAN | |

### 7.2 `mon_alert_event`

| Column | Type | Notes |
|---|---|---|
| `rule_id` | UUID | |
| `fingerprint` | VARCHAR(64) | Labels hash |
| `state` | VARCHAR(20) | |
| `started_at` | TIMESTAMPTZ | |
| `acked_by` | UUID NULL | |
| `resolved_at` | TIMESTAMPTZ NULL | |
| `labels` | JSONB | |
| `value` | NUMERIC NULL | |

**Unique active:** `(rule_id, fingerprint)` where state in FIRING/ACKED.

### 7.3 `mon_silence`

| Column | Type | Notes |
|---|---|---|
| `matchers` | JSONB | |
| `starts_at` / `ends_at` | TIMESTAMPTZ | |
| `created_by` | UUID | |
| `ticket_ref` | VARCHAR(100) NULL | |
| `comment` | TEXT | |

### 7.4 `mon_receiver`

| Column | Type | Notes |
|---|---|---|
| `receiver_key` | VARCHAR(50) | |
| `kind` | VARCHAR(20) | |
| `notify_topic_key` | VARCHAR(100) NULL | p15 |
| `webhook_url` | TEXT NULL | |
| `secret_ref_key` | VARCHAR(150) NULL | |

---

## 8. Dashboards & synthetics

### 8.1 `mon_dashboard_version`

| Column | Type | Notes |
|---|---|---|
| `dashboard_id` | UUID | |
| `version_number` | INT | |
| `definition` | JSONB | Grafana-compatible subset |
| `checksum` | VARCHAR(64) | |
| `lifecycle` | VARCHAR(20) | |

### 8.2 `mon_synthetic_check`

| Column | Type | Notes |
|---|---|---|
| `check_key` | VARCHAR(100) UNIQUE | |
| `kind` | VARCHAR(20) | HTTP/API_FLOW |
| `interval_sec` | INT | |
| `locations` | JSONB | |
| `steps` | JSONB | |
| `alert_rule_id` | UUID NULL | |

---

## 9. Incidents (lite)

### 9.1 `mon_incident`

| Column | Type | Notes |
|---|---|---|
| `incident_key` | VARCHAR(50) | |
| `title` | VARCHAR(200) | |
| `severity` | VARCHAR(20) | |
| `status` | VARCHAR(20) | OPEN/ACK/RESOLVED |
| `commander_user_id` | UUID NULL | |
| `started_at` / `resolved_at` | TIMESTAMPTZ | |

Not a full ITSM — bridge to external optional.

---

## 10. Governance & packs

Seed packs:

- `monitoring.api.slo.default@1.0.0`  
- `monitoring.messaging.queues@1.0.0`  
- `monitoring.event_bus.lag@1.0.0`  
- `monitoring.scheduler.ticker@1.0.0`  
- `monitoring.dashboards.core@1.0.0`  

---

## 11. Plumbing

| Table | Purpose |
|---|---|
| `mon_outbox` | Alert/probe events |
| `mon_idempotency_key` | Ack/silence APIs |

---

## 12. RLS summary

| Class | Policy |
|---|---|
| Catalog/rules/dashboards | Ops roles |
| Tenant business metrics | FORCE tenant if present |
| Alert ack | Authenticated ops |
| Backends/secrets | Admin |

---

## 13. Seed minimum

1. Services matching logging catalog  
2. Core metrics: http rate/error/latency, process up, queue depth  
3. Probes LIVE/READY for api  
4. SLO 99.9% API availability 28d + burn policy  
5. Alert routes → notify topic `system.alert`  
6. Runbooks placeholders  
7. Permissions `monitoring.*`  

---

## 14. ER overview

```text
service ── components / dependencies / teams
metric ── labels / scrape / recording
sli ── slo ── burn policies / budgets
probe ── results / status
alert_rule ── events / history
route ── receivers / escalation
silence / inhibit
dashboard versions / synthetics / incidents
packages
```

---

## 15. Implementation notes

1. Alert evaluator worker polls TSDB; does not scrape apps directly (except probes/synthetics).  
2. Fingerprint for alerts = hash(sorted labels).  
3. READY probe may check DB/redis/event_bus lag; LIVE only process up.  
4. Split models: `catalog`, `slo`, `probe`, `trace`, `alert`, `dashboard`, `synthetic`, `governance`, `plumbing`.
