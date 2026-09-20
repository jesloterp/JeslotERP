# Event Bus (`p13_event_bus`)

**Package:** `p13_event_bus`  
**Schema:** `event_bus`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Domain-event control plane: contracts, schema registry, topics, subscriptions, outbox relay, delivery, and DLQ / replay.

## 2. Responsibilities

- Own the `event_bus` persistence schema and the `p13_event_bus` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Type and schema catalog
- Topics, subscriptions, and consumer groups
- Publish, relay, and outbox sources
- Pull / ack / nack, DLQ, and replay APIs
- Grants, retry, and packs
- HTTP publish persists events and outbox when a database session is present

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Event type**
- **Schema version**
- **Topic**
- **Subscription**
- **Consumer group**
- **Publish grant**
- **DLQ**
- **Replay**
- **Outbox source**

## 5. Major Capabilities

- Type and schema catalog
- Topics, subscriptions, and consumer groups
- Publish, relay, and outbox sources
- Pull / ack / nack, DLQ, and replay APIs
- Grants, retry, and packs
- HTTP publish persists events and outbox when a database session is present

## 6. Public Architecture

```text
HTTP / internal API
        ↓
Application services (commands, queries, ports)
        ↓
Domain concepts and policies
        ↓
Adapters (persistence, optional providers, outbox)
```

Conceptual tables (purpose only):

- Types / schemas
- Topology
- Event store / outbox
- Delivery / DLQ
- Relay / governance

Public API resource groups:

- Catalog
- Topology
- Publish / relay
- Delivery / DLQ
- Governance
- Internal ticks

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`

### Additional verified runtime coupling

- Auth contract shared with the metadata plane

### Recommended future dependency

- p14_messaging as transport
- all platforms as producers
- p19_audit as a consumer

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Messaging
- Audit
- Integration
- Licensing bridges

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `event_bus.schema.activated`
- Seeded example contracts exist for document, process, feature, and sample domain types

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Default retry and backoff live in catalog seed data.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- event permissions
- Publish / subscribe grants
- RLS
- Schema validation

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Filter engine
- Handler registration
- Packages / changesets

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `event_bus` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Relay / delivery catalog still has an in-memory double for part of the path
- Live broker integration is via p14_messaging and remains provider-pending
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Business modules must emit facts (posted, cancelled, received) rather than call each other through ORM.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
