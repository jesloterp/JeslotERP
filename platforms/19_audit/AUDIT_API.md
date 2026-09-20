# JeslotERP Audit Platform — Complete API Specification (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — event ingest/query Postgres-first; SIEM `test-connection` is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p19_audit`  
**PostgreSQL schema:** `audit`  
**Public base:** `/api/v1/audit`  
**Internal base:** `/internal/v1/audit`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`AUDIT_GUIDE.md`](AUDIT_GUIDE.md) · [`AUDIT_SCHEMA.md`](AUDIT_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Ingest, query, integrity verify, retention/holds, export, alerts, SIEM, break-glass, access-of-audit, packs. |
| 1.1 | 2026-09-12 | TASK-SOR-017: durable append-only ingest + query fetch. |
| 1.2 | 2026-09-12 | TASK-SOR-024: `POST /siem/endpoints/{endpoint_key}/test-connection`; live HTTPX never invents DELIVERED. |

---

## 1. Design principles (advanced)

1. **Append-only writes** — no patch/delete event APIs.  
2. **Service ingest primary** — users query; services write.  
3. **Redacted by default** — sensitive read is break-glass.  
4. **Idempotent ingest** on `source_event_id`.  
5. **Every sensitive read audited**.  
6. **Export async** with checksummed artifact.  
7. **Purge only via retention jobs**.  
8. **Legal hold blocks purge**.  
9. **Hash verify** available to compliance.  
10. **Query cost guards** — required time window / object scope for broad queries.  
11. **SIEM deliveries signed**.  
12. **Catalog actions** preferred over free-text action names.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Correlation-Id: <uuid>
X-Internal-Token: <token>
X-Break-Glass-Token: <token>   # when elevated
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
AUD_ACTION_UNKNOWN / OBJECT_TYPE_UNKNOWN
AUD_EVENT_NOT_FOUND
AUD_INGEST_DENIED / INGEST_DUPLICATE / INGEST_REJECTED
AUD_QUERY_TOO_BROAD / QUERY_DENIED
AUD_SENSITIVE_DENIED / BREAK_GLASS_REQUIRED / BREAK_GLASS_EXPIRED
AUD_HOLD_ACTIVE / PURGE_BLOCKED
AUD_EXPORT_NOT_READY / EXPORT_FAILED
AUD_SEAL_INVALID / CHAIN_BROKEN
AUD_ALERT_INVALID
AUD_SIEM_UNAVAILABLE
AUD_IDEMPOTENCY_CONFLICT
AUD_PACKAGE_CHECKSUM_MISMATCH
AUD_WRITER_NOT_GRANTED
```

HTTP: `404` · `409` · `422` · `403` · `429`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `audit.write` | Ingest |
| `audit.read` | Redacted query |
| `audit.read.sensitive` | Unredacted / BG |
| `audit.export` | Exports |
| `audit.hold` | Legal holds |
| `audit.retention.manage` | Retention |
| `audit.alert.manage` | Alerts |
| `audit.admin` | Catalog/seals/SIEM |
| `audit.access.read` | Access trail |
| `audit.*` | All |

---

## 6. Ingest APIs (primary for platforms)

### 6.1 Append event

```http
POST /api/v1/audit/events
POST /internal/v1/audit/events
Idempotency-Key: …
```

```json
{
  "action_key": "ENTITY_UPDATE",
  "object_type": "sales.order",
  "object_id": "…",
  "object_display": "BL/MH01/2526/000148",
  "outcome": "SUCCESS",
  "occurred_at": "2026-09-09T04:30:00Z",
  "actor_type": "USER",
  "actor_id": "…",
  "company_id": "…",
  "request_id": "…",
  "correlation_id": "…",
  "session_id": "…",
  "source_platform": "transport",
  "source_event_id": "outbox:…",
  "data": { "status_to": "CANCELLED" },
  "field_changes": [
    { "field_key": "status", "old_value": "POSTED", "new_value": "CANCELLED", "value_type": "STRING" }
  ],
  "tags": ["bilty", "cancel"]
}
```

Server masks fields per policy, assigns stream sequence + hashes.  
Duplicate `source_event_id` → `200` with existing event + `meta.idempotent_replay=true` (or `409 AUD_INGEST_DUPLICATE` per policy — prefer replay 200).

### 6.2 Batch ingest

```http
POST /internal/v1/audit/events:batch
```

Max batch size enforced.

### 6.3 Event-bus ingest tick

```http
POST /internal/v1/audit/ingest/from-event
```

Body = CloudEvent; mapped via `aud_event_binding`.

---

## 7. Query APIs (primary for investigators)

### 7.1 Search events

```http
POST /api/v1/audit/query
```

```json
{
  "from": "2026-09-01T00:00:00Z",
  "to": "2026-09-09T23:59:59Z",
  "action_keys": ["ENTITY_UPDATE", "DOCUMENT_RELEASE"],
  "object_type": "sales.order",
  "object_id": "…",
  "actor_id": null,
  "outcome": null,
  "q": "CANCELLED",
  "cursor": null,
  "size": 50,
  "include_field_changes": true
}
```

Requires bounded time window unless `object_id` present (`AUD_QUERY_TOO_BROAD` otherwise).  
Returns redacted projection; records `aud_access_event`.

### 7.2 Get event

```http
GET /api/v1/audit/events/{event_id}
GET /api/v1/audit/events/{event_id}/field-changes
GET /api/v1/audit/objects/{object_type}/{object_id}/timeline
```

### 7.3 Sensitive / break-glass read

```http
POST /api/v1/audit/break-glass
POST /api/v1/audit/events/{event_id}/sensitive
```

**Break-glass:**

```json
{
  "reason": "Fraud investigation INV-992",
  "ticket_ref": "JIRA-123",
  "ttl_minutes": 60
}
```

Issues short-lived token; all sensitive reads logged.

---

## 8. Integrity

```http
GET  /api/v1/audit/streams
POST /api/v1/audit/streams/{stream_key}/verify
GET  /api/v1/audit/seals
POST /api/v1/audit/seals
GET  /api/v1/audit/verify-runs/{id}
```

**Verify** checks hash chain continuity for range; on failure creates `aud_tamper_alert`.

**Seal:**

```json
{ "stream_key": "tenant:…", "to_seq": 10500 }
```

---

## 9. Retention & legal hold

```http
GET  /api/v1/audit/retention-policies
PUT  /api/v1/audit/retention-policies/{policy_key}/bindings

GET  /api/v1/audit/legal-holds
POST /api/v1/audit/legal-holds
POST /api/v1/audit/legal-holds/{hold_id}/release

POST /api/v1/audit/purge/runs
GET  /api/v1/audit/purge/runs/{id}
```

**Hold:**

```json
{
  "hold_key": "CASE-2026-44",
  "reason": "Litigation",
  "filters": {
    "object_types": ["sales.order", "document.document"],
    "from": "2025-01-01T00:00:00Z"
  }
}
```

Purge blocked with `AUD_PURGE_BLOCKED` / `AUD_HOLD_ACTIVE` when overlapping.

---

## 10. Export

```http
POST /api/v1/audit/exports
GET  /api/v1/audit/exports/{case_id}
POST /api/v1/audit/exports/{case_id}/package
GET  /api/v1/audit/exports/{case_id}/download-url
```

**Create:**

```json
{
  "case_key": "GST-AUDIT-FY26-Q1",
  "query": { "from": "…", "to": "…", "action_categories": ["FINANCIAL"] },
  "format": "JSONL_GZIP"
}
```

Async packaging → `media_id` via p08 + checksum; download via signed URL. Access audited.

---

## 11. Alerts & SIEM

```http
GET  /api/v1/audit/alert-rules
POST /api/v1/audit/alert-rules
GET  /api/v1/audit/alerts
POST /api/v1/audit/alerts/{id}/ack

GET  /api/v1/audit/siem/endpoints
POST /api/v1/audit/siem/endpoints
GET  /api/v1/audit/siem/deliveries
POST /api/v1/audit/siem/endpoints/{endpoint_key}/test-connection
POST /internal/v1/audit/siem/tick
```

**Alert rule example:** action_key in `PERMISSION_GRANT`, `FEATURE_KILL`, `BREAK_GLASS_OPEN` → notify ops.

SIEM `test-connection`: `stub` / `memory` / `test` → STUB `OK` (pytest double). Configured or live keys (`httpx`, `splunk`, `sentinel`, `webhook`, catalog endpoints) → `ok: false`, `status: PROVIDER_PENDING`. Requires `audit.admin` via `require_audit_access`. Pytest tick still uses `StubSiemForwarder`. No live Splunk/Sentinel client in this slice.

---

## 12. Catalog admin

```http
GET  /api/v1/audit/actions
POST /api/v1/audit/actions
GET  /api/v1/audit/object-types
PUT  /api/v1/audit/field-policies/{object_type}
GET  /api/v1/audit/bindings
PUT  /api/v1/audit/bindings
GET  /api/v1/audit/writers
PUT  /api/v1/audit/writers
```

---

## 13. Access trail (audit-of-audit)

```http
GET /api/v1/audit/access?from=…&to=…&actor_id=…
```

Requires `audit.access.read`.

---

## 14. Saved queries

```http
GET  /api/v1/audit/saved-queries
POST /api/v1/audit/saved-queries
POST /api/v1/audit/saved-queries/{id}/execute
```

---

## 15. Packages & governance

```http
GET  /api/v1/audit/packages
POST /api/v1/audit/packages/{package_key}/install
GET  /api/v1/audit/changesets
POST /api/v1/audit/changesets/{id}/approvals
```

---

## 16. Health

```http
GET /api/v1/audit/health
GET /internal/v1/audit/health
```

Ingest lag, seal freshness, purge blockers, chain head age. `siem` is `PROVIDER_PENDING`; `siem_forwarder` is `STUB` under pytest.

---

## 17. Caching & concurrency

| Resource | Strategy |
|---|---|
| Action catalog | Cached |
| Chain head | Row lock per stream on append |
| Ingest idempotency | Unique source key |
| Export | Single packager per case |

---

## 18. Example flows

### 18.1 Bilty cancel

1. Domain updates + outbox  
2. Service `POST /audit/events` ENTITY_UPDATE + field status  
3. Investigator timeline by object_id  

### 18.2 Permission grant alert

1. Identity emits PERMISSION_GRANT  
2. Alert rule fires → notify  
3. Export case for security review  

### 18.3 Tamper check

1. Nightly `verify` on tenant streams  
2. Chain break → tamper alert → freeze purge  

### 18.4 Litigation hold

1. Create legal hold on sales order+docs  
2. Retention purge skips held rows  
3. After release, normal retention resumes  

---

## 19. Event hooks

| Event | Consumer |
|---|---|
| `audit.alert.fired` | Notification / on-call |
| `audit.seal.created` | Compliance archive |
| `audit.tamper.detected` | Security incident |
| `audit.export.completed` | Requester notify |
| `audit.hold.applied` | Compliance |

---

## 20. Compatibility notes

- Public prefix `/api/v1/audit`; schema `audit`.  
- p20 logs are complementary, not substitutes.  
- Field history UIs should call audit timeline, not duplicate history tables per module (except ultra-hot denorm caches).  
- Client-supplied `occurred_at` may be stored as claim but ordering uses ingest sequence for hash chain.

---

## 21. Related documents

- Guide: [`AUDIT_GUIDE.md`](AUDIT_GUIDE.md)  
- Schema: [`AUDIT_SCHEMA.md`](AUDIT_SCHEMA.md)  
- Event bus: [`../13_event_bus/EVENT_BUS_API.md`](../13_event_bus/EVENT_BUS_API.md)  
- Logging: [`../20_logging/LOGGING_API.md`](../20_logging/LOGGING_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
