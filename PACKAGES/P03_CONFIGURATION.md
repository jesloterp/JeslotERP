# Configuration (`p03_configuration`)

**Package:** `p03_configuration`  
**Schema:** `configuration`  
**Layer:** Platform Foundation  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Hierarchical runtime settings, definitions, secret references, templates, and effective-value resolution.

## 2. Responsibilities

- Own the `configuration` persistence schema and the `p03_configuration` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Modules, categories, definitions, and options
- Scoped settings read / write
- Effective resolve across tenant / company / branch
- Secret metadata and rotation (ciphertext; no plaintext public GET)
- Templates
- Dependency policies
- Admin versus tenant APIs
- History and outbox

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Configuration module**
- **Definition**
- **Scoped setting value**
- **Secret reference**
- **Template**
- **Effective resolution stack**

## 5. Major Capabilities

- Modules, categories, definitions, and options
- Scoped settings read / write
- Effective resolve across tenant / company / branch
- Secret metadata and rotation (ciphertext; no plaintext public GET)
- Templates
- Dependency policies
- Admin versus tenant APIs
- History and outbox

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
- Setting values
- Secret ledger
- Templates
- Audit / outbox

Public API resource groups:

- Catalog
- Settings
- Secrets
- Templates
- Effective resolve
- Policies
- Admin / internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`

### Additional verified runtime coupling

- Key-management provider from p28_security for ciphertext wrap (local-dev path verified; enterprise KMS pending)

### Recommended future dependency

- p05_metadata for setting UI descriptors
- p12_feature when a setting is actually a rollout flag

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Most platforms that need scoped runtime values
- Notification provider secret refs
- Dashboard

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `configuration.module.registered`
- `configuration.definition.created`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Scope types include tenant, company, and branch. Secret material is referenced, not embedded in docs.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Secret value never returned on public GET
- Dedicated secret permissions
- Optimistic concurrency on writes

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Definition catalog consumed by other platforms
- Secret-reference indirection

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `configuration` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Live enterprise KMS wrap is not claimed as production-ready
- Registry Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Business modules should store policy values here rather than hard-coding constants.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
