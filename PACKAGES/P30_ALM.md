# ALM (`p30_alm`)

**Package:** `p30_alm`  
**Schema:** `alm`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Application lifecycle and transport: packages, artifacts, seal / sign, export / import, promote, and rollback across landscapes.

## 2. Responsibilities

- Own the `alm` persistence schema and the `p30_alm` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Environments
- Package CRUD and artifacts
- Seal / sign
- Export and import
- Promote and rollback
- Deploy-agent test-connection

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Environment**
- **Transport package**
- **Artifact**
- **Import job**
- **Promotion**
- **Rollback record**

## 5. Major Capabilities

- Environments
- Package CRUD and artifacts
- Seal / sign
- Export and import
- Promote and rollback
- Deploy-agent test-connection

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

- Environments
- Packages / artifacts
- Import / promotion / rollback
- Outbox

Public API resource groups:

- Environments / packages / artifacts / export / import / promote / rollback
- Internal environments

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`

### Additional verified runtime coupling

- Signing path uses p28_security

### Recommended future dependency

- Live deploy agents per landscape

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Metadata / localization / feature / rules pack movement (intended)
- release operators

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `alm.package.created`
- `alm.package.sealed`
- `alm.package.imported`
- `alm.package.promoted`
- `alm.package.rolled_back`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Landscape endpoints are environment-specific and must not be published in public docs.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Seal / sign
- Permissions
- RLS
- Fail-closed live deploy

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Deploy port / HTTP deploy agent

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `alm` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Live landscape deploy agents are fail-closed without configured endpoints
- This is not a Git product

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Metadata, rules, and output templates must move DEV → QA → PROD as sealed packages.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
