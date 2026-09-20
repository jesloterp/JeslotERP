# Metadata (`p05_metadata`)

**Package:** `p05_metadata`  
**Schema:** `metadata`  
**Layer:** Platform Services  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Metadata control plane: data dictionary, UI layouts, validation, field security descriptors, packages, and publish governance.

## 2. Responsibilities

- Own the `metadata` persistence schema and the `p05_metadata` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Dictionary of modules, entities, and fields
- Semantic types
- UI authoring for forms, lists, filters, actions, inspectors, and variants
- User list preferences
- Overlays and packages
- Publish, rollback, and changesets
- Validate API including conditional required / readonly / visible rules
- Effective resolver (layered)
- Field-level security descriptors and redaction helpers
- Drift and pack ETag comparison
- Impact analysis
- Interop / binding hints
- Seeded pilots: bp.partner and org.company

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Module / Entity / Field**
- **UI layout (form, list, filter, action, inspector)**
- **Overlay**
- **Package**
- **Changeset**
- **Published catalog version**
- **Validation rule**
- **Field-level security policy**
- **Binding contract**
- **Effective resolution**

## 5. Major Capabilities

- Dictionary of modules, entities, and fields
- Semantic types
- UI authoring for forms, lists, filters, actions, inspectors, and variants
- User list preferences
- Overlays and packages
- Publish, rollback, and changesets
- Validate API including conditional required / readonly / visible rules
- Effective resolver (layered)
- Field-level security descriptors and redaction helpers
- Drift and pack ETag comparison
- Impact analysis
- Interop / binding hints
- Seeded pilots: bp.partner and org.company

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

- Dictionary
- Semantic layer
- Behavior / validation
- Security descriptors
- UI descriptors
- Packs / overlays
- Governance
- Outbox

Public API resource groups:

- Dictionary
- Effective / UI pack
- Overlays
- Packages
- Governance / publish
- Validate
- UI authoring
- User preferences
- Drift
- Impact
- Security FLS
- Interop / binding

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p03_configuration`

### Additional verified runtime coupling

- Publish / package hooks via p29_extensibility

### Recommended future dependency

- p06_localization for label keys
- p12_feature for channel / flag layers
- p18_search and p24_reporting as catalog consumers

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Identity profile FLS
- Business Partner and Organization layout pilots
- Rules
- Search
- Reporting
- frontend metadata runtime

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `metadata.catalog.published`
- `metadata.catalog.rolled_back`
- `metadata.changeset.submitted`
- `metadata.changeset.approved`
- `metadata.package.installed`
- `metadata.extension.changed`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Effective layers: system, pack, tenant, company, role, user, channel, feature flag.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Metadata permissions
- RLS on catalog access
- FLS descriptors enforced by IAM consumers
- Safe expression AST (no eval, no SQL)

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Package installer
- Layout changeset studio
- Binding contracts
- Extensibility hooks on publish

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `metadata` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Generic entity data façade for arbitrary entities is deferred
- Third business-document entity seed waits on first business module
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Business applications are intended to be mostly metadata + permissions, not one-off screens per document.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
