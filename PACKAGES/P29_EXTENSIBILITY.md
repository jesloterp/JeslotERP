# Extensibility (`p29_extensibility`)

**Package:** `p29_extensibility`  
**Schema:** `extensibility`  
**Layer:** Platform Services  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Allow-listed enhancement / hook pipeline. Isolated execution, no arbitrary code evaluation from the database.

## 2. Responsibilities

- Own the `extensibility` persistence schema and the `p29_extensibility` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Extension points and handlers
- Allow-list
- Binding activate / deactivate
- Execute and execution history
- Built-in sample handlers
- Isolated worker with timeout
- Kernel invoke façade
- Tests cover no-eval and timeout behavior

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Extension point**
- **Handler**
- **Allow-list entry**
- **Binding**
- **Failure policy**
- **Execution**

## 5. Major Capabilities

- Extension points and handlers
- Allow-list
- Binding activate / deactivate
- Execute and execution history
- Built-in sample handlers
- Isolated worker with timeout
- Kernel invoke façade
- Tests cover no-eval and timeout behavior

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

- Points
- Handlers
- Allow-list
- Bindings
- Failure policy
- Executions
- Outbox

Public API resource groups:

- Points / handlers / allow-list / bindings / execute / history
- Internal execute

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`

### Additional verified runtime coupling

- Runtime callers include Business Partner, Metadata publish, and Process lifecycle

### Recommended future dependency

- Broader consumer hook coverage as business modules appear

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- p04_business_partner
- p05_metadata
- p10_process

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `extensibility.point.created`
- `extensibility.binding.created`
- `extensibility.execution`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Handler timeout is enforced by the worker.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Allow-list only
- Subprocess isolation
- No arbitrary code from stored definitions

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Handler port
- Kernel point registry
- Built-in handler registry

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `extensibility` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Not every platform is a hooked consumer yet
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Industry and tenant extensions must hook the kernel, not fork it.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
