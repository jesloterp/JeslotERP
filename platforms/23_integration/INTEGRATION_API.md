# JeslotERP Integration Platform — Complete API Endpoints

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — deliver/get/attempts persist and read Postgres when `session` is `AsyncSession`; empty attempts is `[]`. Adapters stay ports. Not Production.  
**Package:** `platforms.p23_integration`  
**Base path:** `/api/v1/integration`  
**Companion:** [`INTEGRATION_GUIDE.md`](INTEGRATION_GUIDE.md) · [`INTEGRATION_SCHEMA.md`](INTEGRATION_SCHEMA.md)

> Admin/control APIs under `/api/v1/integration/*`.  
> Public/partner **inbound webhooks** under `/api/v1/hooks/...` (documented in §8).  
> Domain business APIs remain in their platforms; they emit events that trigger pipelines.

---

## 0. Conventions

### Headers

| Header | Required | Notes |
|---|---|---|
| `Authorization` | Yes (admin APIs) | Bearer JWT |
| `X-Tenant-Id` | Yes | Tenant scope |
| `X-Correlation-Id` | Recommended | Trace |
| `Idempotency-Key` | Manual deliver / replay | Required |
| Webhook signature | Inbound | Per `verify_mode` |

### Envelope

Admin APIs use `StandardResponse`.  
Webhook ACK may be plain `200` with minimal body per partner contract (adapter-defined).

### Common errors

| HTTP | Code | Meaning |
|---|---|---|
| 401 | `AUTH_REQUIRED` / `WEBHOOK_UNAUTHORIZED` | Bad JWT or signature |
| 403 | `FORBIDDEN` | Permission / partner profile |
| 404 | `NOT_FOUND` | Unknown id |
| 409 | `CONFLICT` / `IDEMPOTENCY_REPLAY` | Duplicate key |
| 422 | `VALIDATION_ERROR` / `MAPPING_FAILED` | Transform errors |
| 423 | `CIRCUIT_OPEN` | Breaker open |
| 429 | `RATE_LIMITED` | Outbound or admin flood |
| 503 | `DEPENDENCY_UNAVAILABLE` | External/vault down |

---

## 1. Connector catalog

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/connectors` | `integration.catalog.read` |
| `GET` | `/api/v1/integration/connectors/{connector_key}` | `integration.catalog.read` |
| `GET` | `/api/v1/integration/connectors/{connector_key}/config-schema` | `integration.catalog.read` |

**Item:** `connector_key`, `name`, `kind`, `auth_modes[]`, `capabilities[]`, `adapter_version`

---

## 2. Connections

### 2.1 CRUD

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/connections` | `integration.connection.manage` |
| `POST` | `/api/v1/integration/connections` | `integration.connection.manage` |
| `GET` | `/api/v1/integration/connections/{connection_id}` | `integration.connection.manage` |
| `PATCH` | `/api/v1/integration/connections/{connection_id}` | `integration.connection.manage` |
| `POST` | `/api/v1/integration/connections/{connection_id}/disable` | `integration.connection.manage` |
| `POST` | `/api/v1/integration/connections/{connection_id}/enable` | `integration.connection.manage` |

**POST body (sketch):**

```json
{
  "connector_key": "rest.json.v1",
  "connection_key": "gst.sandbox",
  "name": "GST IRN Sandbox",
  "env": "SANDBOX",
  "partner_id": null,
  "endpoint": { "base_url": "https://example-gst.test/api" },
  "auth": {
    "auth_mode": "OAUTH2_CC",
    "oauth_token_url": "https://example-gst.test/oauth/token",
    "oauth_client_id": "public-client-id",
    "oauth_client_secret_ref": "vault:tenant/gst/client_secret"
  },
  "config": { "timeout_ms": 15000 }
}
```

**GET never returns secret values** — only refs and non-secret fields.

### 2.2 Health & rotation

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/integration/connections/{connection_id}/probe` | `integration.connection.manage` |
| `POST` | `/api/v1/integration/connections/{connection_id}/rotate-secret` | `integration.connection.manage` |
| `GET` | `/api/v1/integration/connections/{connection_id}/health` | `integration.connection.manage` |

**Probe response:** `{ "ok": true, "latency_ms": 120, "checked_at": "…" }`

### 2.3 Circuit breaker

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/connections/{connection_id}/circuit-breaker` | `integration.connection.manage` |
| `POST` | `/api/v1/integration/connections/{connection_id}/circuit-breaker/reset` | `integration.admin` |

---

## 3. Mappings

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/mappings` | `integration.mapping.manage` |
| `POST` | `/api/v1/integration/mappings` | `integration.mapping.manage` |
| `GET` | `/api/v1/integration/mappings/{mapping_id}` | `integration.mapping.manage` |
| `POST` | `/api/v1/integration/mappings/{mapping_id}/versions` | `integration.mapping.manage` |
| `POST` | `/api/v1/integration/mappings/{mapping_id}/publish` | `integration.mapping.manage` |
| `POST` | `/api/v1/integration/mappings/{mapping_id}/test` | `integration.mapping.manage` |

**Test body:**

```json
{
  "version": 3,
  "input": { "invoice_id": "…", "amount": 1000.00 },
  "expect": { "IrnRequest.DocDtls.DocNo": "INV-1" }
}
```

**Response:** `{ "passed": true, "output": {…}, "errors": [] }`

---

## 4. Pipelines

### 4.1 Define & publish

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/pipelines` | `integration.catalog.read` |
| `POST` | `/api/v1/integration/pipelines` | `integration.pipeline.manage` |
| `GET` | `/api/v1/integration/pipelines/{pipeline_id}` | `integration.catalog.read` |
| `PATCH` | `/api/v1/integration/pipelines/{pipeline_id}` | `integration.pipeline.manage` |
| `POST` | `/api/v1/integration/pipelines/{pipeline_id}/versions` | `integration.pipeline.manage` |
| `POST` | `/api/v1/integration/pipelines/{pipeline_id}/publish` | `integration.pipeline.manage` |
| `POST` | `/api/v1/integration/pipelines/{pipeline_id}/retire` | `integration.pipeline.manage` |

**Version body (sketch):**

```json
{
  "trigger": { "event_type": "document.invoice.posted", "manual_allowed": true },
  "bindings": {
    "connection_id": "…",
    "mapping_id": "…"
  },
  "steps": [
    { "step_key": "map", "ordinal": 1, "step_type": "MAP", "config": { "mapping_id": "…" } },
    { "step_key": "call", "ordinal": 2, "step_type": "CALL", "config": { "operation": "POST /irn" }, "on_error": "RETRY" },
    { "step_key": "emit", "ordinal": 3, "step_type": "EMIT_EVENT", "config": { "event_type": "integration.gst.irn.acked" } }
  ],
  "retry_policy_key": "default.exp5",
  "error_policy": { "max_attempts": 5, "dlq_on_exhaust": true }
}
```

### 4.2 Manual trigger

`POST /api/v1/integration/pipelines/{pipeline_id}/deliver`  
**Permission:** `integration.deliver`  
**Header:** `Idempotency-Key`  

```json
{
  "connection_id": "…",
  "payload": { "invoice_id": "…" },
  "correlation_id": "…"
}
```

**Response:** `{ "outbound_message_id": "…", "status": "PENDING", "job_id": "…" }`

---

## 5. Outbound messages & attempts

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/outbound-messages` | `integration.catalog.read` |
| `GET` | `/api/v1/integration/outbound-messages/{id}` | `integration.catalog.read` |
| `GET` | `/api/v1/integration/outbound-messages/{id}/attempts` | `integration.catalog.read` |
| `POST` | `/api/v1/integration/outbound-messages/{id}/cancel` | `integration.deliver` |
| `GET` | `/api/v1/integration/outbound-messages/{id}/payload` | `integration.catalog.read` |

**List query:** `pipeline_id`, `connection_id`, `status`, `from`, `to`, `correlation_id`, `page`

**Payload:** returns meta + signed media URL when stored; never dumps secrets.

**SoR:** `GET …/attempts` and `GET …/{id}` are Postgres-first. Empty attempt list is `[]` (not the in-memory seed). Deliver + internal deliver dual-write the outbound row and attempts. `adapter` on an attempt is `stub` for REST, or `PROVIDER_PENDING` for unknown kinds. No live GST/SAP/Salesforce HTTP.

### Internal worker

| Method | Path | Auth |
|---|---|---|
| `POST` | `/api/v1/integration/internal/deliver/{outbound_message_id}` | service |
| `POST` | `/api/v1/integration/internal/deliver/{id}/complete` | service |

Worker loads connection secrets, applies mapping, calls adapter, writes attempt, schedules retry or DLQ.

---

## 6. Retry policies & rate limits

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/retry-policies` | `integration.pipeline.manage` |
| `POST` | `/api/v1/integration/retry-policies` | `integration.admin` |
| `GET` | `/api/v1/integration/rate-limit-policies` | `integration.pipeline.manage` |
| `PUT` | `/api/v1/integration/connections/{id}/rate-limit` | `integration.connection.manage` |

---

## 7. Dead-letter queue

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/dlq` | `integration.dlq.manage` |
| `GET` | `/api/v1/integration/dlq/{item_id}` | `integration.dlq.manage` |
| `POST` | `/api/v1/integration/dlq/{item_id}/replay` | `integration.dlq.manage` |
| `POST` | `/api/v1/integration/dlq/{item_id}/discard` | `integration.dlq.manage` |
| `POST` | `/api/v1/integration/dlq/bulk-replay` | `integration.admin` |

**Replay body:**

```json
{
  "mode": "SAME_IDEMPOTENCY",
  "reason": "External outage resolved"
}
```

`mode`: `SAME_IDEMPOTENCY` | `NEW_IDEMPOTENCY` (policy-gated).  
Replay writes `integration_dlq_replay` + p19 audit.

---

## 8. Inbound webhooks

### 8.1 Endpoint admin

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/webhooks` | `integration.webhook.manage` |
| `POST` | `/api/v1/integration/webhooks` | `integration.webhook.manage` |
| `PATCH` | `/api/v1/integration/webhooks/{endpoint_id}` | `integration.webhook.manage` |
| `POST` | `/api/v1/integration/webhooks/{endpoint_id}/rotate-secret` | `integration.webhook.manage` |
| `GET` | `/api/v1/integration/webhooks/{endpoint_id}/receptions` | `integration.webhook.manage` |

**POST create response includes** public URL path + one-time verify secret material if generated (then only ref stored).

### 8.2 Public ingress

`POST /api/v1/hooks/{tenant_slug}/{endpoint_key}`  
**Auth:** signature / mTLS per endpoint — **no JWT required**

**Pipeline:**

1. Resolve endpoint (active)  
2. Verify signature → else `401 WEBHOOK_UNAUTHORIZED`  
3. Idempotency key from header/body path → duplicate returns prior ACK  
4. Persist reception (+ optional media payload)  
5. Enqueue `integration.inbound.process` (p14)  
6. Return partner-expected ACK quickly  

`GET /api/v1/hooks/{tenant_slug}/{endpoint_key}` may support challenge/verify for vendors that require it (connector-specific).

### 8.3 Internal process complete

`POST /api/v1/integration/internal/inbound/{reception_id}/complete`  
Emits normalized p13 event or invokes domain gateway per pipeline.

---

## 9. Partner profiles & reconcile

### 9.1 Profiles

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/partner-profiles` | `integration.partner.manage` |
| `POST` | `/api/v1/integration/partner-profiles` | `integration.partner.manage` |
| `PUT` | `/api/v1/integration/partner-profiles/{id}/pipelines` | `integration.partner.manage` |
| `PUT` | `/api/v1/integration/partner-profiles/{id}/connections` | `integration.partner.manage` |

### 9.2 Reconcile / sync state

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/reconcile` | `integration.catalog.read` |
| `GET` | `/api/v1/integration/reconcile/{canonical_type}/{canonical_id}` | `integration.catalog.read` |
| `PUT` | `/api/v1/integration/reconcile` | `integration.pipeline.manage` |
| `GET` | `/api/v1/integration/sync-state` | `integration.catalog.read` |
| `POST` | `/api/v1/integration/sync-state/{connection_id}/reset-cursor` | `integration.admin` |

**PUT reconcile:**

```json
{
  "connection_id": "…",
  "canonical_type": "Invoice",
  "canonical_id": "…",
  "external_id": "IRN-…",
  "sync_hash": "…"
}
```

---

## 10. Analytics, packs, admin

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/integration/analytics/usage` | `integration.catalog.read` |
| `GET` | `/api/v1/integration/analytics/errors` | `integration.catalog.read` |
| `GET` | `/api/v1/integration/packages` | `integration.admin` |
| `POST` | `/api/v1/integration/packages/{package_key}/apply` | `integration.admin` |
| `GET` | `/api/v1/integration/alert-rules` | `integration.admin` |
| `PUT` | `/api/v1/integration/alert-rules` | `integration.admin` |

**Usage query:** `from`, `to`, `connection_id`, `pipeline_id`, `granularity`

---

## 11. Permission matrix (summary)

| Surface | Min permission |
|---|---|
| Catalog / browse deliveries | `integration.catalog.read` |
| Connections | `integration.connection.manage` |
| Mappings | `integration.mapping.manage` |
| Pipelines | `integration.pipeline.manage` |
| Manual deliver | `integration.deliver` |
| Webhooks admin | `integration.webhook.manage` |
| DLQ | `integration.dlq.manage` |
| Partner profiles | `integration.partner.manage` |
| Packs / breaker reset / cursor reset | `integration.admin` |
| Internal deliver/inbound | service principal |

---

## 12. Example flows

### 12.1 GST IRN on invoice post

1. Domain emits `document.invoice.posted` (p13)  
2. Pipeline trigger matches → create outbound message  
3. p14 runs deliver → map → REST call  
4. Success → reconcile external IRN; emit `integration.gst.irn.acked`  
5. Failure retryable → backoff; exhaust → DLQ + ops alert (p15)

### 12.2 Payment gateway webhook

1. `POST /api/v1/hooks/{tenant}/payments` + HMAC  
2. Reception stored; async process  
3. Mapping → canonical `PaymentCaptured`  
4. Emit p13 → domain settles invoice  
5. Duplicate webhook idempotency → same ACK, no double settle

### 12.3 Operator replay

1. `GET /dlq?status=OPEN`  
2. Inspect attempts/errors  
3. `POST /dlq/{id}/replay` with reason  
4. New/same idempotency per policy; audit recorded

---

## 13. Related documents

- Guide: [`INTEGRATION_GUIDE.md`](INTEGRATION_GUIDE.md)  
- Schema: [`INTEGRATION_SCHEMA.md`](INTEGRATION_SCHEMA.md)  
- Event bus: [`../13_event_bus/EVENT_BUS_API.md`](../13_event_bus/EVENT_BUS_API.md)  
- Messaging: [`../14_messaging/MESSAGING_API.md`](../14_messaging/MESSAGING_API.md)  
- API platform: [`../22_api/API_ENDPOINTS.md`](../22_api/API_ENDPOINTS.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
