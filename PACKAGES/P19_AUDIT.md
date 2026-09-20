# Audit (`p19_audit`)

**Package:** `p19_audit`  
**Schema:** `audit`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Immutable business and security audit trail: ingest, query, integrity, retention, legal hold, export, and optional SIEM forward.

## 2. Responsibilities

- Own the `audit` persistence schema and the `p19_audit` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Ingest, query, timeline, and break-glass
- Streams, seals, retention, holds, export, catalog
- Internal batch ingest and from-event ingest
- Persistence-backed event ingest / query
- SIEM HTTP port exists and is fail-closed without an endpoint

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Audit event**
- **Field change**
- **Stream / chain pointer**
- **Legal hold**
- **Retention policy**
- **Export case**
- **SIEM endpoint**
- **Break-glass**

## 5. Major Capabilities

- Ingest, query, timeline, and break-glass
- Streams, seals, retention, holds, export, catalog
- Internal batch ingest and from-event ingest
- Persistence-backed event ingest / query
- SIEM HTTP port exists and is fail-closed without an endpoint

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

- Catalog / actions / object types
- Events / field changes
- Integrity / ingest
- Retention / holds
- Export / alert
- Governance

Public API resource groups:

- Ingest / query / timeline
- Ops (streams, holds, export, SIEM)
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p13_event_bus`

### Additional verified runtime coupling

- Secret resolver path for SIEM endpoints

### Recommended future dependency

- All platforms as emitters
- p31_privacy for hold coordination
- p08_file_media for export artifacts

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Privacy
- Licensing audit bridge
- future finance and tax inspectors

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `audit.alert.fired`
- `audit.hold.applied`
- `audit.tamper.detected`
- `audit.seal.created`
- `audit.export.completed`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

SIEM attach is optional at startup.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Sensitive-read and break-glass permissions
- Hash-chain verify
- Access-to-audit is itself auditable

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- SIEM forwarder port
- Internal ingest APIs
- Saved queries

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `audit` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Live SIEM endpoint is environment-blocked
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Statutory and security investigations require an immutable trail independent of application logs.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
