# Rules (`p11_rules`)

**Package:** `p11_rules`  
**Schema:** `rules`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Business-rules and decision control plane: decision tables, safe expressions, evaluate / explain, overlays, and simulation.

## 2. Responsibilities

- Own the `rules` persistence schema and the `p11_rules` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Catalog and definition lifecycle
- Decision tables and expressions
- Evaluate, batch, and assign APIs
- Overlays
- Governance, suites, and stats
- Evaluate logs and compiled artifacts persist

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Rule definition**
- **Decision table and hit policy**
- **Expression AST**
- **Rule set / agenda**
- **Overlay**
- **Eval log / explain**
- **Simulation suite**
- **Compiled artifact**

## 5. Major Capabilities

- Catalog and definition lifecycle
- Decision tables and expressions
- Evaluate, batch, and assign APIs
- Overlays
- Governance, suites, and stats
- Evaluate logs and compiled artifacts persist

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
- Definitions / sources
- Tables / expressions
- Sets / facts
- Runtime logs
- Governance
- Outbox

Public API resource groups:

- Catalog
- Definitions
- Evaluate / simulate
- Overlays
- Suites / governance
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p03_configuration`
- `p05_metadata`

### Additional verified runtime coupling

- Auth contract shared with the metadata plane

### Recommended future dependency

- p10_process gateway conditions
- p16_cache for compiled artifacts
- p19_audit for decision evidence

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Process (recommended)
- Metadata validation (related AST safety bar)
- future pricing, credit, and tax decision callers

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `rules.definition.published`
- `rules.definition.activated`
- `rules.overlay.changed`
- `rules.simulation.failed`
- `rules.pack.installed`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

AST allow-lists and evaluation budgets.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- rules permissions
- RLS
- Allow-listed AST only
- Separate apply permission

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Overlays
- Action catalog (caller executes side effects)
- Package install

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `rules` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Evaluate compute is not exclusively on compiled artifacts yet
- Full DMN product depth is not claimed
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Credit limits, tax applicability, and approval thresholds should be data, not scattered conditionals.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
