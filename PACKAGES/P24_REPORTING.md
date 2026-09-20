# Reporting (`p24_reporting`)

**Package:** `p24_reporting`  
**Schema:** `reporting`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Report and dataset catalog: compile, run, export, subscriptions, sharing, and snapshots.

## 2. Responsibilities

- Own the `reporting` persistence schema and the `p24_reporting` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Folders, catalog, favorites
- Datasets with versions, RLS, budget, preview, binding
- Reports: compile, runs, exports, variants, parameters
- Subscriptions, share links, snapshots
- Admin purge / budgets
- Warehouse SQL compilation path
- Internal run complete / subscription fire

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Folder**
- **Dataset**
- **Report definition**
- **Run**
- **Export**
- **Subscription**
- **Snapshot**
- **Share grant**
- **Query budget**

## 5. Major Capabilities

- Folders, catalog, favorites
- Datasets with versions, RLS, budget, preview, binding
- Reports: compile, runs, exports, variants, parameters
- Subscriptions, share links, snapshots
- Admin purge / budgets
- Warehouse SQL compilation path
- Internal run complete / subscription fire

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

- Folders
- Datasets
- Reports
- Execution / export / subscription
- Security / governance

Public API resource groups:

- Catalog / datasets / reports / runs / exports / subscriptions / shares / snapshots
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p02_organization`
- `p05_metadata`
- `p18_search`

### Additional verified runtime coupling

- Optional warehouse engine attach

### Recommended future dependency

- p08_file_media for export artifacts
- p14_messaging for runs

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Dashboard
- future finance and operations reports

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `reporting.dataset.published`
- `reporting.run.started`
- `reporting.run.completed`
- `reporting.export.ready`
- `reporting.subscription.fired`
- `reporting.snapshot.sealed`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Warehouse engine selection occurs at module startup when configured.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Dataset RLS evaluator
- Reporting permissions
- Fail-closed compile

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Warehouse engine port
- Dataset / report compilers

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `reporting` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Enterprise warehouse attachment is environment-dependent
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Statutory and operational reports must be definitions, not one-off SQL in application code.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
