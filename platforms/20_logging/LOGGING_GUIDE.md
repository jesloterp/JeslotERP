# JeslotERP Logging Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — hot ingest/query persists on Postgres; empty catalog is `[]`. External sinks are `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p20_logging`  
**PostgreSQL schema:** `logging`  
**Depends on:** `p01_identity`  
**Integrates with:** all platforms (emitters), `p03_configuration` (sink secrets), `p12_feature`, `p14_messaging`, `p16_cache`, `p17_scheduler`, `p19_audit` (distinct), `p21_monitoring` (alerts/metrics from log signals)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`LOGGING_SCHEMA.md`](LOGGING_SCHEMA.md) · [`LOGGING_API.md`](LOGGING_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise logging plane: structured schema, correlation, pipelines/sinks, sampling, PII scrub, retention, fingerprints, short-term query store, shipper config, alert hooks. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-018: hot records + catalog HTTP Postgres-first; empty list is `[]`; `require_logging_access` sets RLS GUCs. STDOUT sink is in-process; OTLP/ELASTIC/FILE stay `PROVIDER_PENDING`. |

---

## 1. Purpose (enterprise)

`p20_logging` is JeslotERP’s **structured operational logging control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Application Log (SLG1) + technical logging discipline** — categorized, correlatable messages  
- **Microsoft Dynamics / Azure Monitor & App Insights** — traces, dependency logs, correlation  
- **Salesforce debug / event log file governance** — levels, retention, access control  
- **OpenTelemetry + ELK/Loki class stacks** — pipelines, processors, exporters  

It is **not** `print()` / unstructured files without policy. It is the system that makes ERP operability correct for:

1. **Structured JSON logs** with stable fields  
2. **Correlation** — `request_id`, `trace_id`, `span_id`, `correlation_id`, `tenant_id`  
3. **Levels & categories** — DEBUG→FATAL with per-service overrides  
4. **Pipelines** — collect → scrub → sample → route → sink  
5. **Multi-sink export** — stdout, OTLP, Elastic/Loki, CloudWatch, SIEM  
6. **PII scrubbing** before egress  
7. **Sampling & rate limits** to protect cost/noise  
8. **Error fingerprints** for deduped incident signals  
9. **Short-term queryable store** (hot) + long-term sink retention  
10. **Clear separation from compliance audit (p19)**  

### Owns

| Domain | Examples |
|---|---|
| Log schema & field catalog | required/optional attributes |
| Sources / services | api, workers, tickers |
| Levels & overrides | per env/service/tenant |
| Pipelines & processors | scrub, enrich, sample |
| Sinks / exporters | OTLP, ES, files |
| Shipper configs | agent manifests |
| Hot store | recent logs for UI/API |
| Fingerprints | error signatures |
| Retention | hot/cold policies |
| Access control | who may query logs |

### Does **not** own

| Concern | Owner |
|---|---|
| Immutable compliance trail | `p19_audit` |
| Metrics, traces UI, SLO burn | `p21_monitoring` (consumes log-derived signals) |
| Business domain events | `p13_event_bus` |
| Secret storage | `p03_configuration` |

### Critical split: Logging vs Audit vs Monitoring

| | **Logging (p20)** | **Audit (p19)** | **Monitoring (p21)** |
|---|---|---|---|
| Question | What happened technically? | Who changed what (compliance)? | Is the system healthy? |
| Mutability | Rotatable / TTL | Immutable | Time-series |
| Example | `Timeout talking to SES` | `USER X released DOCUMENT Y` | `p99 latency > 2s` |

**Rule:** Never use logs as the only proof of a regulated business action — emit **p19** too.

---

## 2. Architectural position

```text
App / Worker SDK
    │  structured log record
    ▼
Agent / sidecar / library buffer
    │
    ▼
p20 pipeline (scrub → sample → enrich)
    │
    ├─► hot store (short query)
    ├─► sinks (OTLP/ES/Loki/Cloud…)
    └─► fingerprint → p21 alert hook (optional)
```

**Hard rules**

1. Production default level ≥ INFO (DEBUG via time-boxed override).  
2. Scrub before external sinks.  
3. Tenant_id on business-request logs when available.  
4. No cross-schema FKs.  
5. Query APIs are privileged — logs may contain residual sensitive data.  
6. High-cardinality labels forbidden in metric-bound fields.

---

## 3. Advanced design principles

1. **Schema-first** — required envelope fields.  
2. **OpenTelemetry-aligned** where practical (`trace_id`/`span_id`).  
3. **Category taxonomy** — `http`, `db`, `security`, `messaging`, `provider`, `business_tech`.  
4. **Context propagation** — middleware injects correlation.  
5. **Dynamic level overrides** — service/env/tenant with expiry.  
6. **Sampling** — head/tail/error-biased.  
7. **Rate limit per service** — drop with overflow counters.  
8. **Processors chain** — ordered, allow-listed.  
9. **Sink fan-out** with per-sink filters.  
10. **Fingerprinting** — normalize stack → hash.  
11. **Redaction processors** — PAN/GSTIN/email/token patterns.  
12. **Hot store** — hours/days for investigation UI.  
13. **Cold sinks** own long retention.  
14. **Access audited** lightly (who queried).  
15. **PII classes** on fields.  
16. **CQRS** — ingest path ultra-fast; query separate.  
17. **Backpressure** — drop DEBUG first.  
18. **Packs** — default pipelines per env.

---

## 4. Core concepts

### 4.1 Log record (envelope)

```text
timestamp, level, message,
service, component, env,
tenant_id?, company_id?, user_id?,
request_id, correlation_id?, trace_id?, span_id?,
category, event_name?,
attrs{},          # structured
error?: { type, message, stack, fingerprint },
resource?: { host, version, region }
```

### 4.2 Levels

`TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`

### 4.3 Pipeline

```text
receive → parse → validate schema → scrub → enrich → sample → route → sinks/hot
```

### 4.4 Fingerprint

Normalized error signature for grouping: `service|error_type|top_frames_hash`.

### 4.5 Overrides

Time-boxed DEBUG for `service=notify-worker` or `tenant=…` during incident.

---

## 5. Integration patterns

| Emitter | Practice |
|---|---|
| FastAPI middleware | request start/end + errors |
| p14 workers | job_id, handler_key, attempt |
| p17 ticker | tick_id, shard |
| Providers | outbound latency/errors (no secrets) |

SDK ships to local agent or HTTP ingest.

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `logging.ingest` | Push logs (services) |
| `logging.query` | Query hot store |
| `logging.level.manage` | Overrides |
| `logging.pipeline.manage` | Pipelines/sinks |
| `logging.admin` | Packs, backends |
| `logging.access.read` | Who queried |
| `logging.*` | Wildcard |

### RLS

Hot-store rows with tenant_id FORCE RLS for query.  
Global infra logs (no tenant) require admin.

---

## 7. Module layout

```text
platforms/p20_logging/
  application/
    services/
      ingest.py
      schema_validate.py
      scrubber.py
      sampler.py
      router.py
      fingerprint.py
      query.py
      override.py
    sdk/  # python logging handler
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… sinks/ agents/
  tests/unit/scrub/ sample/ fingerprint/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `logging.override.created` / `expired` | Levels |
| `logging.sink.unhealthy` | Ops |
| `logging.fingerprint.new` | First seen error |
| `logging.pipeline.changed` | Catalog |
| `logging.drop.rate_high` | Backpressure |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, schema catalog, permissions |
| P2 | Ingest API + validate + stdout sink |
| P3 | Scrub + sampling |
| P4 | Hot store query |
| P5 | OTLP/ES sinks + health |
| P6 | Level overrides + fingerprints |
| P7 | Agent manifests + packs |
| P8 | p21 alert hooks |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [ ] Required envelope fields enforced  
- [ ] Scrub removes token-like strings in tests  
- [ ] Sampling reduces DEBUG volume under load test  
- [ ] Override expires automatically  
- [ ] Fingerprint groups identical stacks  
- [ ] Tenant RLS on hot query  
- [ ] Sink secret never in responses  
- [ ] No cross-schema FKs  
- [ ] Clear docs that audit ≠ logging  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Log secrets/JWT/passwords | Scrub + never log |
| Unbounded DEBUG in prod | Overrides with TTL |
| Use logs as SOX evidence | p19 audit events |
| High-cardinality user emails as metric labels | attrs only; careful indexing |
| String-concat logs without structure | Structured fields |
| One infinite local file as SoR | Pipelines + sinks + retention |

---

## 12. Related documents

- Schema: [`LOGGING_SCHEMA.md`](LOGGING_SCHEMA.md)  
- API: [`LOGGING_API.md`](LOGGING_API.md)  
- Audit: [`../19_audit/AUDIT_GUIDE.md`](../19_audit/AUDIT_GUIDE.md)  
- Monitoring: [`../21_monitoring/MONITORING_GUIDE.md`](../21_monitoring/MONITORING_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
