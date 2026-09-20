# Business Platform Overview

## Intent

JeslotERP's kernel is a platform. The business platform is the set of
horizontal ERP modules that will post documents, hold stock, and close books.

Ambition is a compact **enterprise** suite — **without** claiming that suite
exists today, and without affiliation to any named third-party ERP.

## Rules

1. Package name: `business.bNN_<name>`.
2. PostgreSQL schema = domain name (`finance`, `inventory`, …), never `b01`.
3. Business modules consume platforms via API, gateway, and events only.
4. No industry verticals in the current registry (`b18+` reserved for later).
5. Do not implement business code inside `pNN_*`.

## Status summary

| Family | Modules | Code status |
|---|---|---|
| Finance cluster | finance, controlling, tax, treasury, assets, projects | PLANNED |
| Logistics cluster | inventory, warehouse, logistics, quality, maintenance | PLANNED |
| Commercial cluster | sales, purchasing, CRM, service | PLANNED |
| People | HCM | PLANNED |
| Manufacturing | manufacturing | PLANNED |
| Cross | reporting (business content on `p24_reporting`) | PLANNED |

## First vertical (undecided)

Implementation must not start all seventeen modules at once. The kernel
backlog leaves **first business module** as a product decision. Reasonable
defaults:

1. `b01_finance` (chart of accounts + journal + period) if the goal is a
   posting spine.
2. `b05_sales` plus a thin receivable invoice if the goal is order-to-cash.
3. `b07_inventory` if the goal is stock accuracy first.

Until that decision is recorded, every business module remains `PLANNED`.
