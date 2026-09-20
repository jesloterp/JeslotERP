# Platform Overview

**Product:** JeslotERP

JeslotERP is a platform for building multi-tenant ERP and adjacent enterprise
applications. It already provides a numbered kernel of thirty-three packages.
It does **not** yet provide implemented finance, inventory, or sales ledgers.


| Status | Meaning |
|---|---|
| `IMPLEMENTED` | Shipped kernel: HTTP, persistence, permissions, and tests exist for the documented slice. |
| `PARTIALLY_IMPLEMENTED` | Kernel exists, but important engines, providers, or integrations remain incomplete or fail-closed. |
| `FOUNDATION_AVAILABLE` | Supporting contracts, schemas, or hooks exist; the capability is not a complete product. |
| `PLANNED` | Specified in the architecture / business registry; no implementation package found. |
| `TODO` | Explicit work item required before the capability can be called implemented. |
| `FUTURE` | Intended evolution; not current scope. |
| `NOT_FOUND` | No implementation evidence in the current project. |
| `NOT_VERIFIED` | Mentioned in architecture but not independently confirmed as working end-to-end. |


## Problem

ERP products typically mix three things that should be separate:

1. **Who the actor is** and what they may do.
2. **What the business object looks like** and how the UI should behave.
3. **What the industry document means** (invoice, stock move, production order).

When those collapse into one codebase, every new document type becomes a fork.

## Solution shape

JeslotERP keeps (1) and (2) in the platform kernel and reserves (3) for
`business/bNN_*` modules that do not yet exist.

```text
Operator / integrator
        │
        ▼
Versioned HTTP API  (/api/v1) + internal mesh (/internal/v1)
        │
        ▼
33 platform packages (p01–p33)
        │
        ▼
PostgreSQL domain schemas + optional providers (mail, object storage, search, …)
```

## What the kernel already provides

| Concern | Package | Status |
|---|---|---|
| Sign-in, roles, sessions | `p01_identity` | IMPLEMENTED |
| Tenant / company / fiscal | `p02_organization` | IMPLEMENTED |
| Settings and secret refs | `p03_configuration` | IMPLEMENTED |
| Customers / vendors | `p04_business_partner` | IMPLEMENTED |
| Dictionary + UI metadata | `p05_metadata` | IMPLEMENTED |
| Locales and messages | `p06_localization` | IMPLEMENTED |
| Document numbers | `p07_number_series` | IMPLEMENTED |
| Files | `p08_file_media` | PARTIALLY_IMPLEMENTED |
| DMS | `p09_document` | PARTIALLY_IMPLEMENTED |
| Approvals / inbox | `p10_process` | PARTIALLY_IMPLEMENTED |
| Decision tables | `p11_rules` | PARTIALLY_IMPLEMENTED |
| Flags | `p12_feature` | PARTIALLY_IMPLEMENTED |
| Domain events | `p13_event_bus` | PARTIALLY_IMPLEMENTED |
| Jobs | `p14_messaging` | PARTIALLY_IMPLEMENTED |
| Notifications | `p15_notification` | PARTIALLY_IMPLEMENTED |
| Cache policy | `p16_cache` | PARTIALLY_IMPLEMENTED |
| Cron / calendars | `p17_scheduler` | PARTIALLY_IMPLEMENTED |
| Search | `p18_search` | PARTIALLY_IMPLEMENTED |
| Audit trail | `p19_audit` | PARTIALLY_IMPLEMENTED |
| Technical logs | `p20_logging` | PARTIALLY_IMPLEMENTED |
| Metrics / SLO | `p21_monitoring` | PARTIALLY_IMPLEMENTED |
| API products / keys | `p22_api` | PARTIALLY_IMPLEMENTED |
| External connectors | `p23_integration` | PARTIALLY_IMPLEMENTED |
| Reports / datasets | `p24_reporting` | PARTIALLY_IMPLEMENTED |
| Boards / widgets | `p25_dashboard` | PARTIALLY_IMPLEMENTED |
| Entitlements | `p26_licensing` | PARTIALLY_IMPLEMENTED |
| Assistants / RAG | `p27_ai` | PARTIALLY_IMPLEMENTED |
| KMS / posture | `p28_security` | PARTIALLY_IMPLEMENTED |
| Allow-listed hooks | `p29_extensibility` | IMPLEMENTED |
| Transport packages | `p30_alm` | PARTIALLY_IMPLEMENTED |
| Consent / DSR | `p31_privacy` | PARTIALLY_IMPLEMENTED |
| Print / render | `p32_output` | PARTIALLY_IMPLEMENTED |
| Record ACL | `p33_sharing` | PARTIALLY_IMPLEMENTED |

## What it does not yet provide

- General ledger, stock ledger, sales order, purchase order, or tax engine implementations.
- A Production registry label.
- Guaranteed live SaaS vendors (those adapters fail closed without credentials).
- An open-source copy of the implementation.

## How a business application is intended to be built

1. Register entities and layouts in `p05_metadata`.
2. Allocate numbers through `p07_number_series`.
3. Store parties in `p04_business_partner`.
4. Attach files through `p08_file_media` and document identity through `p09_document`.
5. Start approvals in `p10_process`; evaluate decisions in `p11_rules`.
6. Emit facts on `p13_event_bus`.
7. Guard commercial access with `p26_licensing` and `p12_feature`.
8. Keep ledgers inside `business/bNN_*` — never inside `pNN_*`.

That last step is still a roadmap item. See [BUSINESS_PLATFORM/](BUSINESS_PLATFORM/README.md).
