# JeslotERP Logging Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — ingest/query/catalog HTTP persists on Postgres; empty list is `[]`. Sink test/health are honest ports. Not Production.  
**Package:** `platforms.p20_logging`  
**PostgreSQL schema:** `logging`  
**Public base:** `/api/v1/logging`  
**Internal base:** `/internal/v1/logging`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token` / ingest token  
**Companion:** [`LOGGING_GUIDE.md`](LOGGING_GUIDE.md) · [`LOGGING_SCHEMA.md`](LOGGING_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Ingest, query/tail, level overrides, pipelines/sinks, fingerprints, shippers, packs. |
| **1.0 SoR-Live** | **2026-09-12** | Ingest/query/catalog are Postgres-first. `POST /sinks/{key}/test` returns `PROVIDER_PENDING` for external kinds. |

---

## 1. Design principles (advanced)

1. **Ingest is hot path** — batch, buffered, non-blocking for apps.  
2. **Query is privileged** — not a general end-user API.  
3. **Scrub before sink egress**.  
4. **DEBUG via time-boxed override only** in prod.  
5. **Schema validation** on ingest (soft vs hard mode).  
6. **Tenant filter enforced** on query when tenant context present.  
7. **Fingerprints** for ERROR+ automatic.  
8. **No sink secrets in responses**.  
9. **Tail cursors** for live ops.  
10. **Drop metrics** exposed when rate-limited.  
11. **Access to queries recorded**.  
12. **Audit ≠ logging** — don’t offer compliance guarantees here.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
X-Internal-Token: <token>
X-Log-Ingest-Token: <token>     # alternative for agents
X-Tenant-Id: <uuid>
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": { "request_id": "…", "dropped": 0, "accepted": 100 }
}
```

---

## 4. Errors

```text
LOG_SCHEMA_INVALID / LEVEL_INVALID / SERVICE_UNKNOWN
LOG_INGEST_DENIED / RATE_LIMITED / PAYLOAD_TOO_LARGE
LOG_QUERY_DENIED / QUERY_TOO_BROAD / TENANT_REQUIRED
LOG_OVERRIDE_CONFLICT / OVERRIDE_EXPIRED
LOG_PIPELINE_INVALID / SINK_UNHEALTHY
LOG_FINGERPRINT_NOT_FOUND
LOG_SHIPPER_NOT_FOUND
LOG_TAIL_CURSOR_INVALID
LOG_PACKAGE_CHECKSUM_MISMATCH
LOG_IDEMPOTENCY_CONFLICT
```

HTTP: `404` · `409` · `422` · `403` · `413` · `429` · `503`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `logging.ingest` | Ingest |
| `logging.query` | Query/tail |
| `logging.level.manage` | Overrides |
| `logging.pipeline.manage` | Pipelines/sinks |
| `logging.admin` | Packs/backends |
| `logging.access.read` | Query access log |
| `logging.*` | All |

---

## 6. Ingest APIs (primary for services/agents)

### 6.1 Ingest batch

```http
POST /internal/v1/logging/ingest
POST /api/v1/logging/ingest
```

```json
{
  "resource": {
    "service": "api",
    "version": "1.8.0",
    "env": "production",
    "host": "api-1"
  },
  "records": [
    {
      "timestamp": "2026-09-09T04:41:00.123Z",
      "level": "ERROR",
      "message": "SES send timeout",
      "category": "provider",
      "tenant_id": "…",
      "request_id": "…",
      "trace_id": "…",
      "span_id": "…",
      "attrs": {
        "provider": "ses",
        "delivery_id": "…",
        "latency_ms": 5001
      },
      "error": {
        "type": "TimeoutError",
        "message": "timed out",
        "stack": "…"
      }
    }
  ]
}
```

**Response:** `{ accepted, dropped, duplicates, warnings[] }`.  
Always scrubbed before sink; invalid records may soft-drop with warning in non-strict mode.

### 6.2 OTLP-compatible ingest (optional)

```http
POST /internal/v1/logging/otlp
```

Translates OTLP logs to internal schema.

---

## 7. Query & tail (ops)

### 7.1 Query hot store

```http
POST /api/v1/logging/query
```

```json
{
  "from": "2026-09-09T03:00:00Z",
  "to": "2026-09-09T04:41:00Z",
  "levels": ["ERROR", "WARN"],
  "service": "api",
  "tenant_id": "…",
  "request_id": null,
  "trace_id": null,
  "q": "SES",
  "fingerprint": null,
  "size": 100,
  "cursor": null
}
```

Requires time bound ≤ policy max (e.g. 24h) unless `request_id`/`trace_id` provided.  
Records `log_query_access`.

### 7.2 Get by request / trace

```http
GET /api/v1/logging/requests/{request_id}
GET /api/v1/logging/traces/{trace_id}
```

### 7.3 Live tail

```http
POST /api/v1/logging/tail
GET  /api/v1/logging/tail/{cursor_id}
```

```json
{
  "services": ["worker-media"],
  "levels": ["ERROR", "INFO"],
  "poll_seconds": 5
}
```

Returns cursor for next poll (SSE optional later).

---

## 8. Level overrides

```http
GET  /api/v1/logging/overrides
POST /api/v1/logging/overrides
POST /api/v1/logging/overrides/{id}/cancel
```

```json
{
  "level": "DEBUG",
  "expires_at": "2026-09-09T06:00:00Z",
  "reason": "Investigate notify worker",
  "scopes": {
    "env": "production",
    "service_key": "worker-notify"
  }
}
```

SDKs/shippers poll overrides:

```http
GET /internal/v1/logging/config/effective?service=worker-notify&env=production
```

---

## 9. Fingerprints

```http
GET  /api/v1/logging/fingerprints
GET  /api/v1/logging/fingerprints/{fingerprint}
POST /api/v1/logging/fingerprints/{fingerprint}/ack
POST /api/v1/logging/fingerprints/{fingerprint}/resolve
GET  /api/v1/logging/fingerprints/{fingerprint}/hits
```

---

## 10. Pipelines, scrub, sinks

```http
GET  /api/v1/logging/pipelines
POST /api/v1/logging/pipelines
PUT  /api/v1/logging/pipelines/{pipeline_key}/steps
POST /api/v1/logging/pipelines/{pipeline_key}/simulate

GET  /api/v1/logging/scrub-rules
PUT  /api/v1/logging/scrub-rules/{rule_key}

GET  /api/v1/logging/sinks
POST /api/v1/logging/sinks
POST /api/v1/logging/sinks/{sink_key}/test
GET  /api/v1/logging/sinks/{sink_key}/health
PUT  /api/v1/logging/sink-routes
```

**Simulate:** run sample records through scrub/sample without exporting.

---

## 11. Shippers / agents

```http
GET  /api/v1/logging/shippers
POST /api/v1/logging/shippers/register
POST /internal/v1/logging/shippers/{shipper_key}/heartbeat
GET  /internal/v1/logging/shippers/{shipper_key}/manifest
```

Manifest includes pipeline version, sink endpoints (with secret refs resolved locally), sample policies.

---

## 12. Signals (to monitoring/notify)

```http
GET  /api/v1/logging/signal-rules
POST /api/v1/logging/signal-rules
GET  /api/v1/logging/signals
```

Example: new fingerprint on `service=api` severity ERROR → create p21 alert / notify.

---

## 13. Quotas & stats

```http
GET /api/v1/logging/stats/volume?from=…&to=…
GET /api/v1/logging/stats/drops
GET /api/v1/logging/quotas
PUT /api/v1/logging/quotas/tenants/{tenant_id}
```

---

## 14. Access log

```http
GET /api/v1/logging/access?from=…&to=…
```

Requires `logging.access.read`.

---

## 15. Catalog

```http
GET /api/v1/logging/fields
GET /api/v1/logging/services
POST /api/v1/logging/services
GET /api/v1/logging/categories
GET /api/v1/logging/schema
```

---

## 16. Packages & governance

```http
GET  /api/v1/logging/packages
POST /api/v1/logging/packages/{package_key}/install
GET  /api/v1/logging/changesets
POST /api/v1/logging/changesets/{id}/approvals
```

---

## 17. Health

```http
GET /api/v1/logging/health
GET /internal/v1/logging/health
```

Ingest accept rate, sink health, shipper staleness, hot store lag, drop rate.

---

## 18. Caching & concurrency

| Resource | Strategy |
|---|---|
| Effective level config | Cached by service; short TTL + push invalidate |
| Scrub rules | Cached in agents |
| Ingest | Lock-free buffers; overflow drops DEBUG first |
| Hot partitions | Time-based; purge jobs |

---

## 19. Example flows

### 19.1 API request error

1. Middleware logs ERROR with request_id/trace_id/tenant  
2. Ingest → scrub → fingerprint → hot + OTLP  
3. Ops queries by request_id  

### 19.2 Incident DEBUG

1. Create override DEBUG 2h for `worker-notify`  
2. Agents pick effective config  
3. Override expires → back to INFO  

### 19.3 PII scrub

1. Attr accidentally contains phone  
2. Scrub rule redacts before ES sink  
3. Simulate confirms in admin UI  

### 19.4 Log storm

1. Rate limit drops excess INFO  
2. `logging.drop.rate_high` meta event  
3. ERROR still kept via always_levels  

---

## 20. Event hooks

| Event | Consumer |
|---|---|
| `logging.sink.unhealthy` | On-call |
| `logging.fingerprint.new` | p21 / issue tracker |
| `logging.override.created` | Security awareness |
| `logging.drop.rate_high` | Capacity |

---

## 21. Compatibility notes

- Public prefix `/api/v1/logging`; schema `logging`.  
- Prefer internal ingest from agents; public ingest may be disabled in prod.  
- Correlation fields should match API gateway middleware conventions used across platforms.  
- Do not store compliance evidence solely here — use p19.

---

## 22. Related documents

- Guide: [`LOGGING_GUIDE.md`](LOGGING_GUIDE.md)  
- Schema: [`LOGGING_SCHEMA.md`](LOGGING_SCHEMA.md)  
- Audit: [`../19_audit/AUDIT_API.md`](../19_audit/AUDIT_API.md)  
- Monitoring: [`../21_monitoring/MONITORING_API.md`](../21_monitoring/MONITORING_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
