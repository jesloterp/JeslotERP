# Document (`p09_document`)

**Package:** `p09_document`  
**Schema:** `document`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Document management system of record: identity, versions, checkout, libraries, links, and templates. Bytes remain in p08_file_media.

## 2. Responsibilities

- Own the `document` persistence schema and the `p09_document` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Document directory CRUD
- Versions and check-in / check-out
- Status transitions
- Libraries and folders
- Object links
- Templates and generate hooks
- Extended ECM-style surfaces (ACL, distribution, e-sign hooks) exist as API surface

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Document DIR**
- **Version / revision**
- **Checkout lock**
- **Library / folder**
- **Object link**
- **Template / merge job**
- **ACL / controlled copy**

## 5. Major Capabilities

- Document directory CRUD
- Versions and check-in / check-out
- Status transitions
- Libraries and folders
- Object links
- Templates and generate hooks
- Extended ECM-style surfaces (ACL, distribution, e-sign hooks) exist as API surface

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

- Catalog / types
- Document / version
- Checkout
- Library
- Links
- Templates
- Security
- Outbox

Public API resource groups:

- Documents
- Extended libraries / templates / ACL
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p07_number_series`
- `p08_file_media`

### Additional verified runtime coupling

- Record sharing via p33_sharing
- Media gateway completeness is NOT_VERIFIED as a full production integration

### Recommended future dependency

- p05_metadata
- p10_process for release approvals
- p02_organization context

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Process
- Search
- future transactional business documents

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `document.created`
- `document.updated`
- `document.version.created`
- `document.generated`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Catalog seeds and optional series bindings.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- document permissions
- RLS
- ACL entries
- Record sharing
- Legal hold

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Template / generate pipeline
- Series binding

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `document` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Full enterprise content-management parity is not claimed
- Media gateway end-to-end production path is NOT_VERIFIED
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Business documents (orders, invoices, quality certificates) should link here rather than invent per-module file tables.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
