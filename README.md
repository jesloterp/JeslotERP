# JeslotERP — Public Platform Architecture

**Author:** Chetan Patel  
**Name:** JeslotERP (only this name)  
**License (docs):** Apache 2.0 — see [LICENSE](LICENSE)

This repository is the **public architecture and technical specification**.
It is how a serious investor or developer judges the work in a day.

**Aim:** a Python ERP **operating system** at **enterprise posting
discipline** — then business modules that **post**. Application-first ERPs
already exist; this is an **independent** product, not a clone. Source stays
private until the product is good enough to open. Market funding and
**paid** specialists are how the work moves in **record-winning time**.
Timepass is rejected.

JeslotERP is **not affiliated with** SAP, Microsoft, ERPNext, or Odoo.
Those names appear only as comparison. See [TRADEMARKS.md](TRADEMARKS.md).

Read first:

1. [FOUNDER_AIM.md](FOUNDER_AIM.md) — why this product exists
2. [DELIVERY_SPEED.md](DELIVERY_SPEED.md) + [NINETY_DAY_PLAN.md](NINETY_DAY_PLAN.md) — how fast, in what order
3. [FUNDING_AND_TALENT.md](FUNDING_AND_TALENT.md) — capital and paid owners
4. [DEFINITION_OF_DONE.md](DEFINITION_OF_DONE.md) — what “won” means
5. [FAQ.md](FAQ.md)

The kernel is a modular Python monolith (p01–p33, SoR-Live slice). Business
ledgers `b01`–`b17` are **specified, not coded**. That honesty is part of
the product.

> If you cannot own a bounded context, or you want to start all seventeen
> modules at once, do not apply.

## Why it exists

Enterprise resource-planning systems fail when every module invents its own identity, numbering, workflow, audit, and UI. JeslotERP separates:

1. **Platform foundation** — identity, organization, configuration, parties, security primitives, record sharing.
2. **Platform services** — metadata, rules, process, documents, numbering, localization, licensing, AI, ALM, privacy, output, extensibility.
3. **Infrastructure planes** — events, jobs, cache, search, audit, logging, monitoring, API product, integration.
4. **Business platform** — finance, inventory, sales, purchase, and the remaining ERP modules, numbered separately as `bNN_*`.

Business modules are **not** a second copy of the kernel. They consume platforms through APIs, gateways, and events.

## Architecture at a glance

```text
Application / Desk UI
        ↓
Business Platform (b01–b17)     ← PLANNED (registry only)
        ↓
Platform Services (metadata, process, rules, documents, …)
        ↓
Platform Foundation (identity, organization, configuration, partners, sharing, security)
        ↓
Infrastructure (events, jobs, cache, search, observability, API, integration)
```

See [ARCHITECTURE.md](ARCHITECTURE.md) and [PLATFORM_OVERVIEW.md](PLATFORM_OVERVIEW.md).

## Thirty-three platform packages

| # | Package | Name | Status |
|---|---|---|---|
| 01 | `p01_identity` | Identity | IMPLEMENTED |
| 02 | `p02_organization` | Organization | IMPLEMENTED |
| 03 | `p03_configuration` | Configuration | IMPLEMENTED |
| 04 | `p04_business_partner` | Business Partner | IMPLEMENTED |
| 05 | `p05_metadata` | Metadata | IMPLEMENTED |
| 06 | `p06_localization` | Localization | IMPLEMENTED |
| 07 | `p07_number_series` | Number Series | IMPLEMENTED |
| 08 | `p08_file_media` | File / Media | PARTIALLY_IMPLEMENTED |
| 09 | `p09_document` | Document | PARTIALLY_IMPLEMENTED |
| 10 | `p10_process` | Process | PARTIALLY_IMPLEMENTED |
| 11 | `p11_rules` | Rules | PARTIALLY_IMPLEMENTED |
| 12 | `p12_feature` | Feature | PARTIALLY_IMPLEMENTED |
| 13 | `p13_event_bus` | Event Bus | PARTIALLY_IMPLEMENTED |
| 14 | `p14_messaging` | Messaging | PARTIALLY_IMPLEMENTED |
| 15 | `p15_notification` | Notification | PARTIALLY_IMPLEMENTED |
| 16 | `p16_cache` | Cache | PARTIALLY_IMPLEMENTED |
| 17 | `p17_scheduler` | Scheduler | PARTIALLY_IMPLEMENTED |
| 18 | `p18_search` | Search | PARTIALLY_IMPLEMENTED |
| 19 | `p19_audit` | Audit | PARTIALLY_IMPLEMENTED |
| 20 | `p20_logging` | Logging | PARTIALLY_IMPLEMENTED |
| 21 | `p21_monitoring` | Monitoring | PARTIALLY_IMPLEMENTED |
| 22 | `p22_api` | API | PARTIALLY_IMPLEMENTED |
| 23 | `p23_integration` | Integration | PARTIALLY_IMPLEMENTED |
| 24 | `p24_reporting` | Reporting | PARTIALLY_IMPLEMENTED |
| 25 | `p25_dashboard` | Dashboard | PARTIALLY_IMPLEMENTED |
| 26 | `p26_licensing` | Licensing | PARTIALLY_IMPLEMENTED |
| 27 | `p27_ai` | AI | PARTIALLY_IMPLEMENTED |
| 28 | `p28_security` | Security | PARTIALLY_IMPLEMENTED |
| 29 | `p29_extensibility` | Extensibility | IMPLEMENTED |
| 30 | `p30_alm` | ALM | PARTIALLY_IMPLEMENTED |
| 31 | `p31_privacy` | Privacy | PARTIALLY_IMPLEMENTED |
| 32 | `p32_output` | Output | PARTIALLY_IMPLEMENTED |
| 33 | `p33_sharing` | Sharing | PARTIALLY_IMPLEMENTED |

Every package has a public summary under [PACKAGES/](PACKAGES/README.md) and accepted deep specs under [platforms/](platforms/README.md) (GUIDE / SCHEMA / API / RTM).

**Kernel posture for p01–p33:** system-of-record live for the shipped HTTP slice. **None of the thirty-three packages carry a Production registry label.** Soak evidence, signed threat review, and live vendor credentials remain open.

## Metadata-driven approach

`p05_metadata` is the contract catalog: entities, fields, layouts, validation, field-security descriptors, and publish governance. A metadata runtime can render lists and forms from published UI packs.

**Current capability:** dictionary, layered effective resolution, validation (including conditional rules), field-level security descriptors, layout persistence, changeset studio APIs, and seeded pilots for `bp.partner` and `org.company`.

**Target capability:** generic routes for any seeded entity, a full studio experience, and business documents that ship as metadata + process + numbers rather than one-off screens.

See [METADATA_PLATFORM.md](METADATA_PLATFORM.md).

## Business platform vision

Seventeen horizontal ERP modules are registered (`b01_finance` … `b17_service`). **Requirement specifications are finished** in [BUSINESS_PLATFORM/REQUIREMENTS/](BUSINESS_PLATFORM/REQUIREMENTS/README.md). **No business package implementation was found.** Desk tiles are not ledgers.

Recommended first implementation: `b01_finance`. Planning method: [BUSINESS_PLATFORM/HOW_TO_PLAN.md](BUSINESS_PLATFORM/HOW_TO_PLAN.md).

## Extensibility

`p29_extensibility` provides allow-listed hooks with isolated execution. Tenants add fields through metadata overlays. Industry content is intended to travel as sealed ALM packages. Arbitrary `eval` of stored code is out of scope.

See [EXTENSIBILITY.md](EXTENSIBILITY.md).

## Current status (honest)

| Area | Status |
|---|---|
| Platform packages p01–p33 | Present; kernel SoR-Live |
| Production label | Withheld |
| Live vendors (mail, object storage, search cluster, payments, KMS, models, …) | Ports exist; activation is environment-blocked |
| Metadata pilots | Partner and company |
| Business module requirements b01–b17 | Finished specifications |
| Business module code b01–b17 | PLANNED / NOT_FOUND |
| Open-source source release | FUTURE — only if quality justifies it |
| Funding / paid contributors | Intended; not a volunteer-only hobby |

## Documentation map

| Area | Start here |
|---|---|
| Architecture | [ARCHITECTURE.md](ARCHITECTURE.md) |
| Principles and vision | [PLATFORM_PRINCIPLES.md](PLATFORM_PRINCIPLES.md), [PLATFORM_VISION.md](PLATFORM_VISION.md) |
| Packages | [PACKAGE_CATALOG.md](PACKAGE_CATALOG.md), [PACKAGES/](PACKAGES/README.md) |
| Dependencies | [DEPENDENCY_ARCHITECTURE.md](DEPENDENCY_ARCHITECTURE.md) |
| Security / tenancy | [SECURITY_ARCHITECTURE.md](SECURITY_ARCHITECTURE.md), [MULTI_TENANCY.md](MULTI_TENANCY.md) |
| Founder aim | [FOUNDER_AIM.md](FOUNDER_AIM.md) |
| 90-day record sprint | [NINETY_DAY_PLAN.md](NINETY_DAY_PLAN.md) |
| Funding and paid talent | [FUNDING_AND_TALENT.md](FUNDING_AND_TALENT.md) |
| Definition of done | [DEFINITION_OF_DONE.md](DEFINITION_OF_DONE.md) |
| FAQ | [FAQ.md](FAQ.md) |
| Delivery speed (record time) | [DELIVERY_SPEED.md](DELIVERY_SPEED.md) |
| Market position | [MARKET_POSITION.md](MARKET_POSITION.md) |
| Trademarks (third-party names) | [TRADEMARKS.md](TRADEMARKS.md) |
| Deep platform specs | [platforms/](platforms/README.md) |
| Business requirements | [BUSINESS_PLATFORM/REQUIREMENTS/](BUSINESS_PLATFORM/REQUIREMENTS/README.md) |
| Business roadmap | [BUSINESS_PLATFORM/](BUSINESS_PLATFORM/README.md) |
| Open-source readiness | [ROADMAP/OPEN_SOURCE_ROADMAP.md](ROADMAP/OPEN_SOURCE_ROADMAP.md) |
| Contributor guides | [DEVELOPMENT/](DEVELOPMENT/DEVELOPMENT_GUIDE.md) |
| Public / private boundary | [DEVELOPMENT/PUBLICATION_GUIDE.md](DEVELOPMENT/PUBLICATION_GUIDE.md) |

## Status vocabulary


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


## License

Documentation in this tree is licensed under the **Apache License 2.0**. Copyright **2026 Chetan Patel**. See [LICENSE](LICENSE) and [LICENSE.md](LICENSE.md). The private implementation is **not** included and is **not** licensed by those files.

Third-party product names (SAP, Microsoft Dynamics, ERPNext, Odoo, and others)
are marks of their owners. JeslotERP is independent. See [TRADEMARKS.md](TRADEMARKS.md).

## What this repository is not

- It is not a dump of the private source tree.
- It is not a claim that JeslotERP is already an open-source product.
- It is not a customer deployment guide and does not contain credentials, hosts, or tenant data.
- It is not a timepass or unpaid-entertainment project. Serious, funded, paid work is the plan.
