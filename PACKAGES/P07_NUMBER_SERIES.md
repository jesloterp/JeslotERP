# Number Series (`p07_number_series`)

**Package:** `p07_number_series`  
**Schema:** `number_series`  
**Layer:** Platform Services  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Governed document and master numbering: definitions, scopes, allocate / reserve / commit / void, including gapless legal modes.

## 2. Responsibilities

- Own the `number_series` persistence schema and the `p07_number_series` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Catalog of objects, definitions, and segments
- Scope assignments
- Runtime allocate, peek, reserve, commit, and void
- Gap scan, rollover, and buffer operations
- Legal-policy guard for gapless series
- Threshold monitoring
- Package install

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Series object**
- **Definition**
- **Scope assignment**
- **Counter / interval**
- **Allocation ledger**
- **Reservation**
- **Gapless versus buffered mode**

## 5. Major Capabilities

- Catalog of objects, definitions, and segments
- Scope assignments
- Runtime allocate, peek, reserve, commit, and void
- Gap scan, rollover, and buffer operations
- Legal-policy guard for gapless series
- Threshold monitoring
- Package install

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

- Catalog
- Definitions
- Counters
- Allocations
- Legal / governance
- Outbox

Public API resource groups:

- Catalog admin
- Scope assignments
- Runtime allocation
- Admin operations

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p03_configuration`

### Additional verified runtime coupling

- None beyond declared module dependencies.

### Recommended future dependency

- p05_metadata for object catalog binding
- p11_rules for conditional series selection

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Business Partner codes
- Document platform
- all future transactional business modules

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `number_series.number.issued`
- `number_series.number.reserved`
- `number_series.number.voided`
- `number_series.interval.exhausted`
- `number_series.threshold.breached`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Allocation mode, segment templates, and threshold policies.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- number_series permissions
- RLS
- Legal policy guard

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Package installer
- Simulator and gap-scanner operations

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `number_series` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Multi-node gapless race soak evidence is still required before a Production label

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Invoices, orders, journals, and statutory documents require unique, auditable numbers.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
