# JeslotERP Event Bus Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — HTTP/internal publish writes `eb_event` + `eb_outbox` in one commit when `session` is `AsyncSession`. Relay/delivery catalog remains an in-memory double. Not Production.  
**Package:** `platforms.p13_event_bus`  
**PostgreSQL schema:** `event_bus`  
**Depends on:** `p01_identity`  
**Integrates with:** all platforms (outbox producers), `p14_messaging` (transport workers), `p03_configuration`, `p12_feature`, `p16_cache`, `p19_audit`, `p20_logging`, `p21_monitoring`, `p23_integration`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`EVENT_BUS_SCHEMA.md`](EVENT_BUS_SCHEMA.md) · [`EVENT_BUS_API.md`](EVENT_BUS_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise event mesh: CloudEvents contracts, schema registry, topics/subscriptions, outbox relay standard, consumer registry, filters, ordering, retries/DLQ, replay, poison handling, deliveries, packs. |
| 1.1 | 2026-09-12 | TASK-SOR-015: HTTP publish + internal publish persist event+outbox same commit; RLS on publish/event reads. |

---

## 1. Purpose (enterprise)

`p13_event_bus` is JeslotERP’s **domain event control plane & outbox relay standard** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Event Mesh / Enterprise Event Enablement** — event types, topics, subscriptions, CloudEvents  
- **Microsoft Dynamics / Azure Event Grid + Service Bus** — typed events, filters, retries, DLQ  
- **Salesforce Platform Events / CDC** — publish/subscribe business events with schema  
- **Transactional Outbox pattern** — reliable emission after DB commit  

It is **not** “call Redis pub/sub from a request handler.” It is the system that makes ERP integration correct for:

1. **Canonical event contracts** (`bilty.cancelled.v1`, `document.released.v1`)  
2. **CloudEvents 1.0 envelopes** with tenant/company extensions  
3. **Schema registry** (JSON Schema / Avro-ready) with compatibility rules  
4. **Transactional outbox** standards every platform must follow  
5. **Relay** from platform outboxes → bus topics  
6. **Consumer registry** — who subscribes, with filters & concurrency  
7. **At-least-once delivery** + consumer idempotency keys  
8. **Ordering** by partition/business key where required  
9. **Retry, backoff, DLQ, poison** handling  
10. **Replay** from retention window for recovery / new consumers  

### Owns

| Domain | Examples |
|---|---|
| Event type catalog | names, versions, owners |
| Schemas | payload schemas + compatibility |
| Topics / channels | routing topology |
| Subscriptions | consumer bindings + filters |
| Outbox standards | envelope, status, relay cursors |
| Delivery ledger | attempts, acks, failures |
| DLQ / poison | quarantine & redrive |
| Replay | cursor ranges |
| AuthZ | publish/subscribe permissions |
| Observability hooks | metrics/trace correlation |
| Governance | publish schema, packs |

### Does **not** own

| Concern | Owner |
|---|---|
| Generic job queues / delayed workers | `p14_messaging` |
| Notification email/SMS content | `p15_notification` (may consume events) |
| Business mutate logic | Domain consumers |
| Platform-local outbox tables | Each platform (`identity_outbox`, `doc_outbox`, …) — **relayed via p13 contracts** |
| External partner webhooks | `p23_integration` (subscribes/bridges) |

### Critical split: Event Bus vs Messaging vs Integration

| | **Event Bus (p13)** | **Messaging (p14)** | **Integration (p23)** |
|---|---|---|---|
| Focus | Domain event contracts & delivery semantics | Queues, workers, delayed jobs | External systems |
| Payload | Business events (CloudEvents) | Commands/jobs often | Mapped external formats |
| Producers | Platforms via outbox | Workers/API | Connectors |

**Rule:** Domain facts → outbox (same TX) → p13 relay → subscribers. Never dual-write event after commit without outbox.

---

## 2. Architectural position

```text
Platform TX
  ├─ business tables
  └─ platform_*_outbox  ──relay──►  p13 topics
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
               p10 process      p15 notify        p23 integration
               p18 search       p19 audit         custom consumers
```

**Hard rules**

1. **Outbox in the same DB transaction** as business state.  
2. Event `type` + `dataschema` must be registered before production publish.  
3. Consumers must be **idempotent** (`event_id` / `idempotency_key`).  
4. No cross-schema FKs — UUID refs; payloads JSONB.  
5. Multi-tenant: `tenant_id` extension required on business events.  
6. p14 may transport bytes; p13 owns meaning & ACLs.

---

## 3. Advanced design principles

1. **CloudEvents core** — `id`, `source`, `type`, `specversion`, `time`, `subject`, `dataschema`, `data`.  
2. **JeslotERP extensions** — `tenantid`, `companyid`, `correlationid`, `causationid`, `partitionkey`.  
3. **Versioned types** — `sales.order.cancelled.v1` (additive v2).  
4. **Compatibility** — BACKWARD / FORWARD / FULL / NONE on schema evolve.  
5. **Topic taxonomy** — `jesloterp.{domain}.{entity}.{verb}` or type-based routing.  
6. **Subscription filters** — type prefix, tenant, attributes CE.  
7. **Ordering key** — optional strict order per key (costly; explicit).  
8. **Delivery modes** — push (HTTP/worker) & pull (cursor).  
9. **Ack / nack** with visibility timeout for pull.  
10. **Retry policy** — max attempts, exponential backoff, jitter.  
11. **DLQ** after max attempts; manual redrive.  
12. **Poison classification** — schema fail vs handler fail.  
13. **Replay API** — by time/offset for subscription.  
14. **Producer authZ** — `event.publish.{domain}` style permissions.  
15. **Consumer authZ** — subscribe grants per topic/type.  
16. **PII policy** — redact flags on types; audit access to payloads.  
17. **Idempotent relay** — outbox row id → bus event id stable.  
18. **Packs** — seed core platform event contracts.

---

## 4. Core concepts

### 4.1 Event type

```text
type_key = "sales.order.cancelled.v1"
source_pattern = "jesloterp/document/sample"
owner_platform = "transport" | "p09_document" | …
schema_id, compatibility, pii_class, retention_days
```

### 4.2 CloudEvents envelope (example)

```json
{
  "specversion": "1.0",
  "id": "8f1c…",
  "source": "jesloterp/document/sample",
  "type": "sales.order.cancelled.v1",
  "time": "2026-09-09T04:20:00Z",
  "subject": "sales.order/8f1c…",
  "dataschema": "/schemas/sales.order.cancelled.v1",
  "data": { "bilty_id": "…", "reason_code": "CUSTOMER_REQUEST" },
  "tenantid": "…",
  "companyid": "…",
  "correlationid": "…",
  "causationid": "…",
  "partitionkey": "bilty:…"
}
```

### 4.3 Outbox row (platform-local standard)

Every platform outbox MUST support conceptually:

```text
outbox_id, occurred_at, type, aggregate_type, aggregate_id,
payload, headers, tenant_id, partition_key,
status: PENDING|RELAYED|FAILED|DEAD,
relay_attempts, last_error, relayed_event_id
```

p13 relay reads via registered **outbox sources** (connection metadata + query contract), not cross-schema ORM.

### 4.4 Topic & subscription

- **Topic** — logical stream of types  
- **Subscription** — consumer group + filter + delivery endpoint/mode  
- **Consumer group** — competing consumers; each event processed once per group  

### 4.5 Delivery lifecycle

```text
RELAYED → DISPATCHED → (ACKED | NACKED → RETRY → … → DLQ)
```

### 4.6 Ordering

| Mode | Behavior |
|---|---|
| `NONE` | Max throughput |
| `KEY` | Same `partitionkey` ordered per subscription |
| `TYPE_TENANT` | Per type+tenant (rare) |

---

## 5. Reliability model

| Guarantee | Notes |
|---|---|
| Produce | At-least-once from outbox relay |
| Consume | At-least-once; duplicates possible |
| Exactly-once | **Not claimed** — use idempotent handlers |
| Loss | Only if outbox TX missing (bug) or retention expired before replay |

---

## 6. Integration patterns

### 6.1 Platform emit

```text
same TX: update bilty + insert outbox
relay worker: claim PENDING → validate schema → publish to topic → mark RELAYED
```

### 6.2 Consumer

```text
receive → check idempotency store → handle → ack
on failure → nack → retry → DLQ
```

### 6.3 Process / notify

p10/p15 register subscriptions for needed types; handlers call their application services.

---

## 7. Security

### Permissions

| Code | Use |
|---|---|
| `event.catalog.read` | Read types/schemas |
| `event.catalog.manage` | Manage contracts |
| `event.publish` | Manual/admin publish (rare) |
| `event.publish.{domain}` | Domain-scoped publish |
| `event.subscribe.manage` | Manage subscriptions |
| `event.delivery.read` | Inspect deliveries |
| `event.dlq.manage` | Redrive/purge DLQ |
| `event.replay` | Replay |
| `event.admin` | Topology/relay sources |
| `event.audit.read` | Audit |
| `event.*` | Wildcard |

### RLS

FORCE RLS on tenant-scoped delivery logs / subscriptions when tenant-bound.  
System catalog readable with auth.

---

## 8. Module layout

```text
platforms/p13_event_bus/
  application/
    services/
      schema_registry.py
      compatibility.py
      relay_worker.py
      publisher.py
      dispatcher.py
      filter_engine.py
      ordering.py
      retry_policy.py
      dlq.py
      replay.py
      idempotency.py
    commands/… queries/…
    permissions/catalog.py
  domain/cloudevents/ …
  infrastructure/
    http/… persistence/… messaging/ (bridge to p14)
    relays/ outbox_source_adapters/
  tests/unit/schema/ relay/ filter/ ordering/
```

---

## 9. Domain events (meta)

| Event | When |
|---|---|
| `event_bus.type.registered` / `schema.activated` | Catalog |
| `event_bus.subscription.changed` | Topology |
| `event_bus.relay.lag_high` | Ops |
| `event_bus.dlq.enqueued` | Failures |
| `event_bus.replay.started` / `completed` | Replay |

Stream: `jesloterp:event_bus:outbox` (meta — careful recursion; meta events may bypass or use dedicated channel).

---

## 10. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, catalog, permissions |
| P2 | Schema registry + compatibility |
| P3 | Topics/subscriptions + filters |
| P4 | Outbox source registry + relay |
| P5 | Push/pull delivery + ack |
| P6 | Retry/DLQ/redrive |
| P7 | Ordering keys + idempotency store |
| P8 | Replay + lag metrics |
| P9 | Packs (identity/org/doc/process contracts) |
| P10 | Registry → **Live** |

---

## 11. Definition of Done (enterprise)

- [x] CloudEvents validation on relay  
- [x] Incompatible schema activate blocked  
- [x] Outbox same-TX documented + adapter test  
- [x] HTTP `/publish` and internal `/publish` persist `eb_event` + `eb_outbox` on `AsyncSession` (one commit)  
- [x] Duplicate event_id is idempotent (no second outbox row)  
- [x] RLS GUCs on publish and event list/get (`require_event_access`)  
- [x] Duplicate delivery safe with idempotency API  
- [x] DLQ redrive restores to subscription  
- [x] Filter unit tests  
- [x] Ordering test for KEY mode  
- [x] Tenant extension required for business types  
- [x] No cross-schema FKs  
- [x] Relay lag metric exported (in-process lag samples + `/metrics/lag`)  

---

## 12. Anti-patterns

| Don’t | Do |
|---|---|
| Publish after commit without outbox | Outbox in same TX |
| Free-form event type strings in prod | Registered types only |
| Assume exactly-once | Idempotent consumers |
| Put huge blobs in `data` | media_id references |
| Share one queue for unrelated domains blindly | Topics + filters |
| Replay entire bus casually in prod | Scoped replay + auth |

---

## 13. Related documents

- Schema: [`EVENT_BUS_SCHEMA.md`](EVENT_BUS_SCHEMA.md)  
- API: [`EVENT_BUS_API.md`](EVENT_BUS_API.md)  
- Messaging: [`../14_messaging/MESSAGING_GUIDE.md`](../14_messaging/MESSAGING_GUIDE.md)  
- Integration: [`../23_integration/INTEGRATION_GUIDE.md`](../23_integration/INTEGRATION_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
