# JeslotERP Monitoring Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — metrics/scrape HTTP persists on Postgres; empty list is `[]`. Backend test is an honest TSDB port. Not Production.  
**Package:** `platforms.p21_monitoring`  
**PostgreSQL schema:** `monitoring`  
**Public base:** `/api/v1/monitoring`  
**Internal base:** `/internal/v1/monitoring`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`MONITORING_GUIDE.md`](MONITORING_GUIDE.md) · [`MONITORING_SCHEMA.md`](MONITORING_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Status/probes, metrics catalog, SLO/budget, alerts/ack/silence, dashboards, synthetics, service map, eval workers. |
| **1.0 SoR-Live** | **2026-09-12** | GET/POST `/metrics` and scrape-targets are Postgres-first. `POST /backends/{key}/test` returns `PROVIDER_PENDING` for Prometheus. |
| **1.1** | **2026-09-12** | HYG-021: `GET /slos` is canonical. `/slosos` is not a route. |

---

## 1. Design principles (advanced)

1. **Status & alerts first for operators** — `/status`, `/alerts`.  
2. **SLO-native** — prefer burn alerts over raw threshold spam.  
3. **Cardinality guards** on metric registration.  
4. **LIVE ≠ READY**.  
5. **Silence requires ticket/reason**.  
6. **Dashboards versioned** — publish activate.  
7. **Synthetics** produce first-class alert events.  
8. **Ack/resolve idempotent**.  
9. **Secrets never in dashboard JSON responses resolved**.  
10. **Internal scrape/eval** separate from public read APIs.  
11. **Label allow-lists** enforced.  
12. **Production alert edits** may need approval.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
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
MON_SERVICE_NOT_FOUND / METRIC_NOT_FOUND / SLO_NOT_FOUND
MON_LABEL_DENIED / CARDINALITY_EXCEEDED
MON_PROBE_NOT_FOUND / PROBE_FAILED
MON_ALERT_NOT_FOUND / ALERT_STATE_INVALID
MON_SILENCE_INVALID / ROUTE_INVALID
MON_DASHBOARD_NOT_FOUND / SYNTHETIC_NOT_FOUND
MON_BACKEND_UNHEALTHY / QUERY_FAILED
MON_APPROVAL_REQUIRED / PACKAGE_CHECKSUM_MISMATCH
MON_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
MON_BURN_POLICY_INVALID
```

HTTP: `404` · `409` · `422` · `403` · `503`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `monitoring.read` | Status/dashboards |
| `monitoring.metrics.manage` | Catalog |
| `monitoring.slo.manage` | SLOs |
| `monitoring.alert.manage` | Rules |
| `monitoring.alert.ack` | Ack |
| `monitoring.silence.manage` | Silences |
| `monitoring.synthetic.manage` | Synthetics |
| `monitoring.admin` | Backends/packs |
| `monitoring.*` | All |

---

## 6. Status & probes (primary ops)

### 6.1 Platform status

```http
GET /api/v1/monitoring/status
GET /api/v1/monitoring/status/services/{service_key}
```

Returns aggregated HEALTHY/DEGRADED/DOWN with probe summaries and active critical alerts count.

### 6.2 Probes

```http
GET  /api/v1/monitoring/probes
POST /api/v1/monitoring/probes
GET  /api/v1/monitoring/probes/{probe_key}
GET  /api/v1/monitoring/probes/{probe_key}/results
POST /api/v1/monitoring/probes/{probe_key}/run
```

### 6.3 Public/internal health endpoints (app)

Documented contract for services (implemented in each app, registered here):

```http
GET /health/live
GET /health/ready
```

READY may fail when DB/redis/event_bus lag exceeds thresholds; LIVE stays 200 if process up.

---

## 7. Metrics catalog

```http
GET  /api/v1/monitoring/metrics
POST /api/v1/monitoring/metrics
GET  /api/v1/monitoring/metrics/{metric_key}
PUT  /api/v1/monitoring/metrics/{metric_key}/labels
GET  /api/v1/monitoring/scrape-targets
POST /api/v1/monitoring/scrape-targets
```

**Create metric:**

```json
{
  "metric_key": "http_server_request_duration_seconds",
  "metric_type": "HISTOGRAM",
  "unit": "seconds",
  "labels": ["service", "method", "route", "status_class"],
  "owner_service": "api"
}
```

Rejects `user_id`/`email` style labels via guard → `MON_LABEL_DENIED`.

### Instant query proxy (ops)

```http
POST /api/v1/monitoring/query
POST /api/v1/monitoring/query_range
```

Forwards to TSDB with authz; not for unbounded export.

---

## 8. SLO & error budgets

```http
GET  /api/v1/monitoring/slis
POST /api/v1/monitoring/slis
GET  /api/v1/monitoring/slos
POST /api/v1/monitoring/slos
GET  /api/v1/monitoring/slos/{slo_key}/budget
GET  /api/v1/monitoring/slos/{slo_key}/reports
PUT  /api/v1/monitoring/slos/{slo_key}/burn-policy
GET  /api/v1/monitoring/journeys
```

**Budget response:**

```json
{
  "slo_key": "api.availability.28d",
  "objective": 0.999,
  "compliance": 0.9994,
  "budget_remaining": 0.0004,
  "status": "HEALTHY"
}
```

---

## 9. Alerts (primary ops)

### 9.1 Rules

```http
GET  /api/v1/monitoring/alert-rules
POST /api/v1/monitoring/alert-rules
PATCH /api/v1/monitoring/alert-rules/{rule_key}
POST /api/v1/monitoring/alert-rules/{rule_key}/disable
```

### 9.2 Active alerts

```http
GET  /api/v1/monitoring/alerts?state=FIRING
GET  /api/v1/monitoring/alerts/{alert_id}
POST /api/v1/monitoring/alerts/{alert_id}/ack
POST /api/v1/monitoring/alerts/{alert_id}/resolve
```

**Ack:**

```json
{ "comment": "Looking at SES timeouts" }
```

### 9.3 Routing & escalation

```http
GET  /api/v1/monitoring/routes
PUT  /api/v1/monitoring/routes
GET  /api/v1/monitoring/receivers
POST /api/v1/monitoring/receivers
GET  /api/v1/monitoring/escalation-policies
PUT  /api/v1/monitoring/escalation-policies/{key}
```

### 9.4 Silences

```http
GET  /api/v1/monitoring/silences
POST /api/v1/monitoring/silences
POST /api/v1/monitoring/silences/{id}/expire
```

```json
{
  "matchers": [{ "name": "service", "value": "worker-notify", "op": "EQ" }],
  "starts_at": "…",
  "ends_at": "…",
  "ticket_ref": "CHG-221",
  "comment": "Notify provider maintenance"
}
```

---

## 10. Dashboards

```http
GET  /api/v1/monitoring/dashboards
POST /api/v1/monitoring/dashboards
GET  /api/v1/monitoring/dashboards/{key}
POST /api/v1/monitoring/dashboards/{key}/versions
POST /api/v1/monitoring/dashboards/{key}/versions/{n}/publish
```

---

## 11. Synthetics

```http
GET  /api/v1/monitoring/synthetics
POST /api/v1/monitoring/synthetics
GET  /api/v1/monitoring/synthetics/{check_key}/runs
POST /api/v1/monitoring/synthetics/{check_key}/run
```

**Check example:**

```json
{
  "check_key": "api.login_and_status",
  "interval_sec": 60,
  "locations": ["in-west"],
  "steps": [
    { "method": "GET", "url": "https://api…/health/ready", "expect_status": 200 }
  ]
}
```

---

## 12. Service map & traces

```http
GET /api/v1/monitoring/service-map
GET /api/v1/monitoring/trace-policies
PUT /api/v1/monitoring/trace-policies/{key}
GET /api/v1/monitoring/traces/{trace_id}   # proxy deep link / summary
```

Service map returns nodes/edges with error rates.

---

## 13. Incidents (lite)

```http
GET  /api/v1/monitoring/incidents
POST /api/v1/monitoring/incidents
POST /api/v1/monitoring/incidents/{key}/ack
POST /api/v1/monitoring/incidents/{key}/resolve
POST /api/v1/monitoring/incidents/{key}/link-alert
```

---

## 14. Runbooks & capacity

```http
GET /api/v1/monitoring/runbooks
PUT /api/v1/monitoring/runbooks/{key}
GET /api/v1/monitoring/capacity
```

---

## 15. Internal workers

```http
POST /internal/v1/monitoring/probes/tick
POST /internal/v1/monitoring/alerts/eval
POST /internal/v1/monitoring/slo/eval
POST /internal/v1/monitoring/synthetics/tick
POST /internal/v1/monitoring/service-map/rebuild
POST /internal/v1/monitoring/metrics/remote-write   # optional gated
```

Alert eval applies silences/inhibits → notify via p15 on transition to FIRING.

---

## 16. Packages & governance

```http
GET  /api/v1/monitoring/packages
POST /api/v1/monitoring/packages/{package_key}/install
GET  /api/v1/monitoring/changesets
POST /api/v1/monitoring/changesets/{id}/approvals
```

---

## 17. Backends & health

```http
GET  /api/v1/monitoring/backends
POST /api/v1/monitoring/backends/{key}/test
GET  /api/v1/monitoring/health
```

---

## 18. Caching & concurrency

| Resource | Strategy |
|---|---|
| Alert rules / routes | Cached; invalidate on change |
| Probe status | Hot row upsert |
| Alert fingerprint | Unique active firing |
| SLO eval | Periodic worker; snapshot budgets |

---

## 19. Example flows

### 19.1 API latency SLO burn

1. Histogram metrics scraped  
2. SLO eval computes burn  
3. Alert FIRING → notify `system.alert`  
4. On-call acks → investigates via logging request_id exemplars  

### 19.2 Dependency outage

1. READY probe fails on DB  
2. Status DEGRADED; LIVE still up  
3. Orchestrator stops sending traffic (k8s)  

### 19.3 Maintenance silence

1. Create silence for `worker-notify`  
2. Provider blips don’t page  
3. Silence expires  

### 19.4 Synthetic login path

1. Synthetic fails 3x  
2. Creates alert linked to journey `login`  
3. Incident opened  

---

## 20. Event hooks

| Event | Consumer |
|---|---|
| `monitoring.alert.firing` | p15 / pager |
| `monitoring.probe.down` | Status page |
| `monitoring.slo.burn_high` | Release freeze policy |
| `monitoring.synthetic.failed` | Ops |

---

## 21. Compatibility notes

- Public prefix `/api/v1/monitoring`; schema `monitoring`.  
- Align `service_key` with p20 logging services.  
- Trace deep-dive UI may be external; API provides policy + links.  
- Business KPI metrics allowed only with bounded labels and tenant policy.

---

## 22. Related documents

- Guide: [`MONITORING_GUIDE.md`](MONITORING_GUIDE.md)  
- Schema: [`MONITORING_SCHEMA.md`](MONITORING_SCHEMA.md)  
- Logging: [`../20_logging/LOGGING_API.md`](../20_logging/LOGGING_API.md)  
- Notification: [`../15_notification/NOTIFICATION_API.md`](../15_notification/NOTIFICATION_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
