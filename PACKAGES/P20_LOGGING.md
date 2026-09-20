# Logging (`p20_logging`)

**Package:** `p20_logging`  
**Schema:** `logging`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Structured technical logging plane: hot ingest / query / tail, pipelines, scrub rules, sinks, and fingerprints.

## 2. Responsibilities

- Own the `logging` persistence schema and the `p20_logging` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Ingest, query, tail, overrides, fingerprints, pipelines, sinks, services, schema, packs
- Internal ingest and shipper heartbeat
- Hot ingest / query is persistence-backed
- In-process stdout sink is available
- External sinks exist as ports and remain pending without configuration

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Log service / category**
- **Hot record**
- **Tail cursor**
- **Pipeline / scrub rule**
- **Sink / shipper**
- **Fingerprint**
- **Level override**

## 5. Major Capabilities

- Ingest, query, tail, overrides, fingerprints, pipelines, sinks, services, schema, packs
- Internal ingest and shipper heartbeat
- Hot ingest / query is persistence-backed
- In-process stdout sink is available
- External sinks exist as ports and remain pending without configuration

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

- Services / levels
- Pipeline / scrub
- Sinks / shippers
- Hot store / tail
- Fingerprints / signals
- Governance

Public API resource groups:

- Public ingest / query / tail / catalog
- Internal ingest

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`

### Additional verified runtime coupling

- None beyond declared module dependencies.

### Recommended future dependency

- p21_monitoring for signal correlation
- p03_configuration for sink secret refs

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Monitoring (declared)
- operators and support

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `logging.drop.rate_high`
- `logging.fingerprint.new`
- `logging.override.created`
- `logging.sink.unhealthy`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Sink types are catalogued; enablement is environment-specific.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- logging permissions
- Query breadth limits
- Scrub rules
- Access log for queries

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Sink port / adapters

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `logging` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Production OTLP / file shippers are not claimed as live
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Technical logs must not become a second copy of secrets or personal data.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
