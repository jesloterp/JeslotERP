# JeslotERP Monitoring Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — metrics/scrape catalog HTTP persists on Postgres; empty list is `[]`. External TSDB is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p21_monitoring`  
**PostgreSQL schema:** `monitoring`  
**Depends on:** `p20_logging`  
**Integrates with:** all platforms, `p01_identity`, `p03_configuration`, `p12_feature`, `p14_messaging`, `p15_notification`, `p16_cache`, `p17_scheduler`, `p19_audit` (alert ack audit optional), `p13_event_bus`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`MONITORING_SCHEMA.md`](MONITORING_SCHEMA.md) · [`MONITORING_API.md`](MONITORING_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise monitoring plane: metrics catalog, SLI/SLO/error budgets, health probes, trace policies, alerts/routing/silences, dashboards, synthetics, service map, capacity signals. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-019: metric/scrape HTTP Postgres-first; empty list is `[]`; `require_monitoring_access` sets RLS GUCs. Samples stay on MEMORY TsdbEngine. Prometheus/Mimir stay `PROVIDER_PENDING`. |
| **1.1** | **2026-09-12** | HYG-021: SLO list is `GET /slos` only. `/slosos` typo removed from API + code. |

---

## 1. Purpose (enterprise)

`p21_monitoring` is JeslotERP’s **observability & reliability control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Solution Manager / Focused Run / CCMS** — system health, alerts, landscape monitoring  
- **Microsoft Dynamics / Azure Monitor + App Insights** — metrics, availability, alerts, app maps  
- **Salesforce Scale Center / Health Check / monitoring events** — org health & performance signals  
- **Prometheus + Grafana + OTel + PagerDuty-class SRE stacks** — SLOs, burn alerts, dashboards  

It is **not** a few hardcoded `/health` endpoints. It is the system that makes ERP reliability correct for:

1. **Metrics catalog** — RED/USE, business KPIs (ingest rates, queue depth)  
2. **SLIs / SLOs / error budgets** per service & critical user journey  
3. **Health probes** — liveness/readiness/dependency checks  
4. **Distributed tracing policy** — sampling, retention pointers (spans in Tempo/Jaeger/OTel backend)  
5. **Alerting** — rules, multi-window burn, routing, silences, escalation  
6. **Dashboards-as-code** definitions  
7. **Synthetics** — blackbox checks for login/API critical paths  
8. **Service map / dependency graph**  
9. **Capacity & saturation** signals  
10. **Hooks from logging fingerprints** (p20) into incidents  

### Owns

| Domain | Examples |
|---|---|
| Metric definitions | names, labels allow-list, units |
| Time-series registry | backend pointers (Prometheus/Mimir/…) |
| SLI/SLO | objectives, windows, burn |
| Health checks | probes, dependency status |
| Trace policies | sample rates, slow-span rules |
| Alert rules | thresholds, PromQL/OTel queries |
| Alert routing | teams, channels, escalation |
| Silences / mutes | maintenance |
| Dashboards | JSON defs |
| Synthetics | HTTP/browser checks |
| Incidents (lite) | open/ack/resolve meta |
| Governance | packs, approvals |

### Does **not** own

| Concern | Owner |
|---|---|
| Raw structured log bodies | `p20_logging` |
| Compliance who-changed-what | `p19_audit` |
| Actual TSDB storage cluster | Infra (registered as backend) |
| Span payload warehouse | Trace backend (registered) |
| Email/SMS delivery | `p15_notification` |

### Critical split: Monitoring vs Logging vs Audit

| | **Monitoring (p21)** | **Logging (p20)** | **Audit (p19)** |
|---|---|---|---|
| Signal | Aggregates & health | Verbose diagnostics | Legal accountability |
| Example | `http_p99 > 2s` | stack trace of timeout | user released document |
| Action | Page on-call | Debug | Investigate compliance |

---

## 2. Architectural position

```text
Services / workers / tickers
    │ metrics · spans · probe results
    ▼
Collectors (OTel / Prometheus)
    │
    ▼
p21 control plane (catalog, SLO, alert eval)
    │
    ├─► TSDB / Trace backend (infra)
    ├─► alert → notify / pager
    └─► dashboards / status API
```

**Hard rules**

1. **Label cardinality** controlled — no unbounded `user_id` on metrics.  
2. Alert fatigue managed via SLO burn & silences — not raw spam.  
3. Health `/live` vs `/ready` semantics distinct.  
4. No cross-schema FKs.  
5. Secrets for webhook/pager via configuration.  
6. Production alert rule changes may require approval.

---

## 3. Advanced design principles

1. **Catalog-first metrics** — register before scrape/accept.  
2. **RED for services** — Rate, Errors, Duration.  
3. **USE for resources** — Utilization, Saturation, Errors.  
4. **SLO-native alerting** — multi-window burn rates.  
5. **Probe classes** — LIVE, READY, DEPENDENCY, SYNTHETIC.  
6. **Trace sampling policies** — head/tail/error.  
7. **Alert routing by severity & team**.  
8. **Escalation policies** — L1→L2 timelines.  
9. **Silences** with ticket refs.  
10. **Dashboards-as-code** versioned.  
11. **Synthetics** from outside/in for critical journeys.  
12. **Service map** from traces + declared deps.  
13. **Runbooks** linked on alerts.  
14. **Error budget reports** per window.  
15. **Feature-flagged** noisy experimental metrics.  
16. **CQRS HTTP** — thin APIs; eval workers separate.  
17. **Idempotent alert state transitions**.  
18. **Packs** — core platform SLO/alert baselines.

---

## 4. Core concepts

### 4.1 Metric

```text
metric_key, type COUNTER|GAUGE|HISTOGRAM|SUMMARY,
unit, description, label_keys[], owner_service
```

### 4.2 SLI / SLO

```text
sli = successful_requests / total_requests
slo = 99.9% over 28d
error_budget = 1 - slo
burn alerts on 1h/6h windows
```

### 4.3 Health probe

```text
probe_key, kind LIVE|READY|DEPENDENCY,
target service/url, interval, timeout,
success criteria, status OPEN/HEALTHY/DEGRADED/DOWN
```

### 4.4 Alert lifecycle

```text
PENDING → FIRING → ACKED → RESOLVED
              └→ SILENCED (suppressed)
```

### 4.5 Synthetic check

HTTP login + `GET /api/v1/health` + search smoke from probe locations.

---

## 5. Integration patterns

| Source | Signal |
|---|---|
| API middleware | request rate/error/latency histograms |
| p14 | queue depth, lease age, DLQ count |
| p13 | relay lag |
| p17 | ticker stale |
| p16 | cache hit ratio |
| p20 | fingerprint.new → alert candidate |
| DB | connection pool saturation (exporter) |

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `monitoring.read` | Dashboards/status |
| `monitoring.metrics.manage` | Metric catalog |
| `monitoring.slo.manage` | SLOs |
| `monitoring.alert.manage` | Rules/routing |
| `monitoring.alert.ack` | Ack/resolve |
| `monitoring.silence.manage` | Silences |
| `monitoring.synthetic.manage` | Synthetics |
| `monitoring.admin` | Backends/packs |
| `monitoring.*` | Wildcard |

### RLS

Tenant-scoped business metrics (if any) FORCE tenant.  
Most infra metrics are global with admin/ops roles.

---

## 7. Module layout

```text
platforms/p21_monitoring/
  application/
    services/
      metric_registry.py
      slo_evaluator.py
      burn_alert.py
      probe_runner.py
      alert_manager.py
      silence.py
      dashboard_repo.py
      synthetic_runner.py
      service_map.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… workers/
    backends/ prometheus.py tempo.py
  tests/unit/slo/ alert/ probe/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `monitoring.alert.firing` / `resolved` | Alerts |
| `monitoring.slo.burn_high` | Budget |
| `monitoring.probe.down` | Health |
| `monitoring.synthetic.failed` | Synthetics |
| `monitoring.silence.created` | Ops |
| `monitoring.dashboard.published` | Catalog |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, metric catalog, permissions |
| P2 | Health probes + status API |
| P3 | Metric ingest/registry + backend pointer |
| P4 | Alert rules + notify bridge |
| P5 | SLO + burn alerts |
| P6 | Dashboards + silences |
| P7 | Synthetics + service map |
| P8 | Packs for core platforms |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [ ] LIVE vs READY probes behave differently under dependency failure  
- [ ] SLO burn alert fires in unit simulation  
- [ ] High-cardinality label rejected at registry  
- [ ] Silence suppresses notify  
- [ ] Synthetic failure creates FIRING alert  
- [ ] Ack/resolve audited (optional p19)  
- [ ] No secrets in dashboard JSON  
- [ ] No cross-schema FKs  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Alert on raw CPU only | SLO/user-journey signals |
| `user_id` metric label | Bounded labels |
| Single threshold spam | Multi-window burn |
| /health does heavy DB migrations | Separate LIVE/READY |
| Dashboard-only tribal knowledge | Dashboards-as-code + packs |
| Ignore error budget | Report & gate releases |

---

## 12. Related documents

- Schema: [`MONITORING_SCHEMA.md`](MONITORING_SCHEMA.md)  
- API: [`MONITORING_API.md`](MONITORING_API.md)  
- Logging: [`../20_logging/LOGGING_GUIDE.md`](../20_logging/LOGGING_GUIDE.md)  
- Notification: [`../15_notification/NOTIFICATION_GUIDE.md`](../15_notification/NOTIFICATION_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
