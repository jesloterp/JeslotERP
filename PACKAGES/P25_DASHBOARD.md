# Dashboard (`p25_dashboard`)

**Package:** `p25_dashboard`  
**Schema:** `dashboard`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Operational boards: layouts, widgets, bindings, personalization, refresh, sharing, and threshold alerts.

## 2. Responsibilities

- Own the `dashboard` persistence schema and the `p25_dashboard` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Folders, boards, versions, publish
- Layouts and widgets
- Bindings, widget data, refresh
- Filters, presets, drill
- Personalization and home assignment
- Shares and threshold alerts
- Iframe allow-list and admin budgets
- Board catalog is persistence-backed

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Board**
- **Board version**
- **Layout**
- **Widget**
- **Query binding**
- **Refresh job**
- **Filter preset**
- **Personalization**
- **Home assignment**
- **Threshold alert**

## 5. Major Capabilities

- Folders, boards, versions, publish
- Layouts and widgets
- Bindings, widget data, refresh
- Filters, presets, drill
- Personalization and home assignment
- Shares and threshold alerts
- Iframe allow-list and admin budgets
- Board catalog is persistence-backed

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

- Boards
- Layouts / widgets / bindings
- Refresh
- Personalization
- Security / governance

Public API resource groups:

- Boards / layouts / widgets / bindings / refresh / shares / alerts
- Internal refresh

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p03_configuration`
- `p12_feature`
- `p16_cache`
- `p24_reporting`

### Additional verified runtime coupling

- Widget cache namespace via p16_cache

### Recommended future dependency

- Live warehouse samples from p24_reporting

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Operators and tenant home assignment

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `dashboard.board.published`
- `dashboard.widget.refreshed`
- `dashboard.alert.fired`
- `dashboard.share.changed`
- `dashboard.home.assigned`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Cache namespace via p16_cache.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Board-level shares
- Query budgets
- Iframe allow-list

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Widget registry
- Binding resolver
- Layout engine

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `dashboard` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Widget sample data still has an in-process path; live warehouse-only samples are pending
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

KPIs belong on boards bound to reporting datasets, not hardcoded cards on every list page.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
