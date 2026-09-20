# JeslotERP Event Bus Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — public `/api/v1/event-bus`, internal `/internal/v1/event-bus`; `/publish` persists event+outbox on AsyncSession  
**Package:** `platforms.p13_event_bus`  
**PostgreSQL schema:** `event_bus`  
**Public base:** `/api/v1/event-bus`  
**Internal base:** `/internal/v1/event-bus`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`EVENT_BUS_GUIDE.md`](EVENT_BUS_GUIDE.md) · [`EVENT_BUS_SCHEMA.md`](EVENT_BUS_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Catalog/schemas, topics/subscriptions, relay, publish, pull/push delivery, ack/nack, DLQ/redrive, replay, grants, packs. |
| 1.1 | 2026-09-12 | HYG-014: `POST /changesets` is part of the public contract (not EXTRA). |

---

## 1. Design principles (advanced)

1. **Contracts first** — unknown `type` rejected in production relay.  
2. **CloudEvents everywhere** — public/internal payloads conform.  
3. **Outbox is the producer path** — synchronous `/publish` is admin/break-glass only.  
4. **At-least-once + idempotency** — consumers must dedupe.  
5. **Ack explicit** for pull; push uses HTTP 2xx as ack.  
6. **DLQ is visible** — no silent drop after max retries.  
7. **Replay is scoped & authorized**.  
8. **Schema compatibility enforced** on activate.  
9. **Tenant extension required** for business event types.  
10. **Ordering opt-in** — default unordered for throughput.  
11. **Idempotent relay** — stable mapping outbox_row → event_id.  
12. **PII-aware admin reads** — redact by policy.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Correlation-Id: <uuid>
X-Internal-Token: <token>   # internal only
```

CloudEvents content mode also supported:

```http
Content-Type: application/cloudevents+json
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
EB_TYPE_NOT_FOUND / TYPE_INACTIVE / TYPE_EXISTS
EB_SCHEMA_INVALID / COMPATIBILITY_FAILED / SCHEMA_NOT_ACTIVE
EB_TOPIC_NOT_FOUND / SUBSCRIPTION_NOT_FOUND
EB_FILTER_INVALID / ENDPOINT_INVALID
EB_PUBLISH_DENIED / SUBSCRIBE_DENIED
EB_EVENT_NOT_FOUND / DELIVERY_NOT_FOUND
EB_ACK_INVALID / LEASE_EXPIRED / ALREADY_ACKED
EB_DLQ_EMPTY / REDRIVE_CONFLICT
EB_REPLAY_DENIED / REPLAY_RANGE_INVALID
EB_RELAY_SOURCE_NOT_FOUND / RELAY_LAG_HIGH
EB_ORDERING_BLOCKED
EB_PAYLOAD_TOO_LARGE / TENANT_REQUIRED
EB_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
EB_PACKAGE_CHECKSUM_MISMATCH
EB_PII_DENIED
```

HTTP: `404` · `409` · `422` · `403` · `413` · `412`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `event.catalog.read` | Types/schemas/topics |
| `event.catalog.manage` | Mutate contracts |
| `event.publish` | Admin publish |
| `event.publish.{domain}` | Domain publish grants |
| `event.subscribe.manage` | Subscriptions |
| `event.delivery.read` | Inspect deliveries |
| `event.dlq.manage` | DLQ/redrive |
| `event.replay` | Replay |
| `event.admin` | Relay sources/topology |
| `event.audit.read` | Audit |
| `event.*` | All |

---

## 6. Catalog — event types & schemas

### 6.1 Event types

```http
GET    /api/v1/event-bus/types
POST   /api/v1/event-bus/types
GET    /api/v1/event-bus/types/{type_key}
PATCH  /api/v1/event-bus/types/{type_key}
POST   /api/v1/event-bus/types/{type_key}/retire
```

**Create:**

```json
{
  "type_key": "sales.order.cancelled.v1",
  "domain_key": "transport",
  "source_pattern": "jesloterp/document/sample",
  "require_tenant": true,
  "require_partition_key": true,
  "pii_class": "LOW",
  "retention_days": 30
}
```

### 6.2 Schemas

```http
GET    /api/v1/event-bus/types/{type_key}/schemas
POST   /api/v1/event-bus/types/{type_key}/schemas
GET    /api/v1/event-bus/schemas/{schema_id}
POST   /api/v1/event-bus/schemas/{schema_id}/validate
POST   /api/v1/event-bus/schemas/{schema_id}/publish
POST   /api/v1/event-bus/schemas/{schema_id}/activate
```

**Create schema:**

```json
{
  "format": "JSON_SCHEMA",
  "compatibility": "BACKWARD",
  "content": { "type": "object", "required": ["bilty_id"], "properties": { "bilty_id": { "type": "string", "format": "uuid" } } },
  "examples": [{ "bilty_id": "00000000-0000-0000-0000-000000000001", "reason_code": "CUSTOMER_REQUEST" }]
}
```

**Validate payload:**

```http
POST /api/v1/event-bus/types/{type_key}/validate
```

```json
{ "data": { "bilty_id": "…", "reason_code": "CUSTOMER_REQUEST" } }
```

Activate runs compatibility check against previous ACTIVE → `EB_COMPATIBILITY_FAILED` on break without policy override.

---

## 7. Topics & subscriptions

### 7.1 Topics

```http
GET    /api/v1/event-bus/topics
POST   /api/v1/event-bus/topics
GET    /api/v1/event-bus/topics/{topic_key}
PUT    /api/v1/event-bus/topics/{topic_key}/types
```

### 7.2 Subscriptions

```http
GET    /api/v1/event-bus/subscriptions
POST   /api/v1/event-bus/subscriptions
GET    /api/v1/event-bus/subscriptions/{subscription_key}
PATCH  /api/v1/event-bus/subscriptions/{subscription_key}
POST   /api/v1/event-bus/subscriptions/{subscription_key}/disable
```

**Create:**

```json
{
  "subscription_key": "search.index.document.released",
  "topic_key": "jesloterp.document",
  "consumer_group_key": "search-indexers",
  "delivery_mode": "PUSH_WORKER",
  "ordering_mode": "NONE",
  "handler_key": "search.index_on_document_released",
  "filters": [
    { "op": "TYPE_EQ", "value": "document.released.v1" }
  ],
  "retry_policy_key": "default_exponential",
  "max_concurrency": 10
}
```

### 7.3 Consumer groups

```http
GET  /api/v1/event-bus/consumer-groups
POST /api/v1/event-bus/consumer-groups
```

---

## 8. Publish (admin / break-glass)

```http
POST /api/v1/event-bus/publish
Idempotency-Key: …
```

```json
{
  "specversion": "1.0",
  "type": "sales.order.cancelled.v1",
  "source": "jesloterp/document/sample",
  "subject": "sales.order/…",
  "data": { "bilty_id": "…", "reason_code": "CUSTOMER_REQUEST" },
  "tenantid": "…",
  "companyid": "…",
  "partitionkey": "bilty:…"
}
```

Prefer platform outbox + relay. This API requires `event.publish` and still validates schema.

Internal producer (trusted platform service):

```http
POST /internal/v1/event-bus/publish
```

Still subject to publish grants + schema validation.

---

## 9. Outbox relay admin

```http
GET  /api/v1/event-bus/outbox-sources
POST /api/v1/event-bus/outbox-sources
PATCH /api/v1/event-bus/outbox-sources/{source_key}
GET  /api/v1/event-bus/outbox-sources/{source_key}/cursor
GET  /api/v1/event-bus/relay/lag
POST /internal/v1/event-bus/relay/tick
GET  /api/v1/event-bus/relay/batches?source_key=…
```

**Register source:**

```json
{
  "source_key": "p09_document",
  "platform_code": "p09_document",
  "adapter_kind": "PG_TABLE",
  "table_or_endpoint": "document.doc_outbox",
  "claim_batch_size": 100,
  "poll_interval_ms": 1000
}
```

`relay/tick` claims PENDING outbox rows, validates, inserts `eb_event`, fans out deliveries.

---

## 10. Pull delivery API

### 10.1 Pull

```http
POST /api/v1/event-bus/subscriptions/{subscription_key}/pull
```

```json
{ "max_messages": 10, "visibility_timeout_seconds": 30 }
```

**Response:** messages with `delivery_id`, CloudEvents payload, `lease_expires_at`.

### 10.2 Ack / Nack

```http
POST /api/v1/event-bus/deliveries/{delivery_id}/ack
POST /api/v1/event-bus/deliveries/{delivery_id}/nack
```

**Nack:**

```json
{ "reason": "HANDLER_ERROR", "retryable": true, "detail": "timeout talking to search" }
```

### 10.3 Idempotency helper

```http
POST /api/v1/event-bus/consumer-groups/{group_key}/idempotency
GET  /api/v1/event-bus/consumer-groups/{group_key}/idempotency/{event_id}
```

```json
{ "event_id": "…", "result_hash": "optional" }
```

Returns whether already processed; used by handlers for exactly-once *effect*.

---

## 11. Push delivery (worker / HTTP)

### 11.1 Internal dispatch tick

```http
POST /internal/v1/event-bus/dispatch/tick
```

Pushes due deliveries to worker handlers / HTTP endpoints.

### 11.2 Worker callback

```http
POST /internal/v1/event-bus/deliveries/{delivery_id}/result
```

```json
{ "status": "ACKED" }
```
or
```json
{ "status": "NACKED", "retryable": false, "error_code": "SCHEMA_INVALID" }
```

### 11.3 HTTP push contract

Bus POSTs `application/cloudevents+json` with headers:

```http
X-Eb-Delivery-Id: …
X-Eb-Signature: sha256=…
X-Eb-Attempt: 3
```

Receiver responds `2xx` = ack; `410` = non-retryable; other 5xx/429 = retryable nack.

---

## 12. Inspect events & deliveries

```http
GET /api/v1/event-bus/events/{event_id}
GET /api/v1/event-bus/events?type=document.released.v1&tenant_id=…&from=…&to=…
GET /api/v1/event-bus/deliveries?subscription_key=…&status=DLQ
GET /api/v1/event-bus/deliveries/{delivery_id}
GET /api/v1/event-bus/deliveries/{delivery_id}/attempts
```

Payload may be redacted when `pii_class` high unless `event.audit.read` + policy.

---

## 13. DLQ & redrive

```http
GET  /api/v1/event-bus/subscriptions/{subscription_key}/dlq
POST /api/v1/event-bus/subscriptions/{subscription_key}/dlq/redrive
POST /api/v1/event-bus/dlq/{dlq_id}/discard
GET  /api/v1/event-bus/redrive-jobs/{id}
```

**Redrive:**

```json
{
  "dlq_ids": ["…"],
  "reset_attempts": true
}
```

Creates new PENDING deliveries (or resets) — does not invent new event ids.

---

## 14. Replay

```http
POST /api/v1/event-bus/subscriptions/{subscription_key}/replay
GET  /api/v1/event-bus/replay-jobs/{id}
POST /api/v1/event-bus/replay-jobs/{id}/cancel
```

```json
{
  "from_time": "2026-09-08T00:00:00Z",
  "to_time": "2026-09-09T00:00:00Z",
  "type_prefix": "document.",
  "max_events": 100000
}
```

Requires `event.replay`. Replayed deliveries marked with replay job id; consumers should still idempotency-check.

---

## 15. Grants, policies, packs

```http
GET  /api/v1/event-bus/grants/publish
PUT  /api/v1/event-bus/grants/publish
GET  /api/v1/event-bus/grants/subscribe
PUT  /api/v1/event-bus/grants/subscribe
GET  /api/v1/event-bus/retry-policies
POST /api/v1/event-bus/retry-policies

GET  /api/v1/event-bus/packages
POST /api/v1/event-bus/packages/{package_key}/install
GET  /api/v1/event-bus/changesets
POST /api/v1/event-bus/changesets
POST /api/v1/event-bus/changesets/{id}/approvals
```

`POST /changesets` is a shipped governance route (HYG-014). It is not an undocumented extra.

---

## 16. Outbox contract (developer standard)

```http
GET /api/v1/event-bus/outbox-contract
POST /api/v1/event-bus/outbox-contract/validate-sample
```

Returns required fields/statuses platforms must implement. Used in CI to validate platform outbox migrations.

---

## 17. Health & metrics

```http
GET /api/v1/event-bus/health
GET /api/v1/event-bus/metrics/lag
GET /api/v1/event-bus/metrics/dlq-counts
GET /internal/v1/event-bus/health
```

---

## 18. Caching & concurrency

| Resource | Strategy |
|---|---|
| Active schemas | Cache by type_key |
| Subscriptions/filters | Cache; invalidate on change |
| Relay claim | `FOR UPDATE SKIP LOCKED` style |
| Pull lease | visibility timeout row lock |
| KEY ordering | per partition_key sequencer |

---

## 19. Example flows

### 19.1 Document released → search

1. p09 TX: status RELEASED + `doc_outbox` row  
2. Relay tick → `document.released.v1` event  
3. Subscription `search.index.document.released` delivery  
4. Handler indexes; idempotency put; ack  

### 19.2 Poison schema

1. Bad payload fails validation at relay → outbox FAILED/DEAD + poison record  
2. Ops fixes producer; redrive outbox (platform tool) or republish  

### 19.3 Consumer bug → DLQ → redrive

1. Handler throws → nack retries exhausted → DLQ  
2. Fix handler  
3. `dlq/redrive` → reprocess  

### 19.4 New subscriber catch-up

1. Create subscription  
2. `replay` last 24h for type prefix  
3. Idempotent handler absorbs duplicates  

---

## 20. Event hooks (meta)

| Event | Consumer |
|---|---|
| `event_bus.dlq.enqueued` | On-call notify |
| `event_bus.relay.lag_high` | Autoscaling / alert |
| `event_bus.schema.activated` | Producer CI cache bust |
| `event_bus.replay.completed` | Ops audit |

---

## 21. Compatibility notes

- Public prefix `/api/v1/event-bus`; schema `event_bus`.  
- Platform outboxes remain in platform schemas; p13 does not own those tables.  
- p14 may carry dispatch jobs; event meaning stays here.  
- Prefer `type` versioning (`.v2`) over incompatible in-place edits.

---

## 22. Related documents

- Guide: [`EVENT_BUS_GUIDE.md`](EVENT_BUS_GUIDE.md)  
- Schema: [`EVENT_BUS_SCHEMA.md`](EVENT_BUS_SCHEMA.md)  
- Messaging: [`../14_messaging/MESSAGING_API.md`](../14_messaging/MESSAGING_API.md)  
- Integration: [`../23_integration/INTEGRATION_API.md`](../23_integration/INTEGRATION_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
