# Business Module Catalog

Canonical numbering from the business registry. Status is evidence-based.

**Finished requirement specs:** [REQUIREMENTS/](REQUIREMENTS/README.md).
Code remains `PLANNED` / `NOT_FOUND` until a module is built on the private kernel.

| Module | Package | Schema | Current status | Platform dependency | Priority | TODO focus |
|---|---|---|---|---|---|---|
| Accounting / Finance | `b01_finance` | `finance` | PLANNED / NOT_FOUND | p01–p03, p07, p05, p10, p11, p19 | High | COA, journal, GL, periods, close |
| Controlling | `b02_controlling` | `controlling` | PLANNED / NOT_FOUND | b01, p02 | Medium | Cost objects, budgets, allocations |
| Tax | `b03_tax` | `tax` | PLANNED / NOT_FOUND | b01, p23 | High | Tax codes, calculate, returns coordination |
| Treasury | `b04_treasury` | `treasury` | PLANNED / NOT_FOUND | b01, b03, p04 | Medium | Banks, payments, cash |
| Sales | `b05_sales` | `sales` | PLANNED / NOT_FOUND | p04, b01, b07, p07, p10 | High | Quote, order, delivery, AR |
| Purchasing | `b06_purchasing` | `purchasing` | PLANNED / NOT_FOUND | p04, b01, b07, p07, p10 | High | PR, PO, GRN, AP |
| Inventory | `b07_inventory` | `inventory` | PLANNED / NOT_FOUND | b01, p02, p07, p11 | High | Items, stock ledger, valuation |
| Warehouse | `b08_warehouse` | `warehouse` | PLANNED / NOT_FOUND | b07, p02 | Medium | Bins, putaway, pick |
| Logistics | `b09_logistics` | `logistics` | PLANNED / NOT_FOUND | b05–b08, p07, p10 | Medium | Transport, consignments, freight |
| Manufacturing | `b10_manufacturing` | `manufacturing` | PLANNED / NOT_FOUND | b07, b02 | Medium | BOM, routing, production orders |
| Quality | `b11_quality` | `quality` | PLANNED / NOT_FOUND | b07, b10 | Low | Inspection, NCR |
| Maintenance | `b12_maintenance` | `maintenance` | PLANNED / NOT_FOUND | b07, b13 | Low | Equipment, PM work orders |
| Fixed assets | `b13_fixed_assets` | `assets` | PLANNED / NOT_FOUND | b01, p17 | Medium | Asset master, depreciation |
| Projects | `b14_projects` | `projects` | PLANNED / NOT_FOUND | b01, b02, b05 | Medium | WBS, costing, billing |
| HCM | `b15_hcm` | `hcm` | PLANNED / NOT_FOUND | p01, p02 | Low | HR ops; payroll interface later |
| CRM | `b16_crm` | `crm` | PLANNED / NOT_FOUND | p04, b05 | Medium | Leads, opportunities, activities |
| Service | `b17_service` | `service` | PLANNED / NOT_FOUND | b16, b12, b07 | Low | Service orders, SLA, field service |
| Reporting (content) | uses `p24_reporting` | `reporting` | FOUNDATION_AVAILABLE (platform) / PLANNED (business content) | p24, p05, b01+ | High | Statutory and operational datasets |
| Industry extensions | `b18+` | n/a | FUTURE | ALM + metadata packs | Low | Not in current registry |

Pilot masters that already exist **on the platform** (not business packages):

| Master | Owner | Status |
|---|---|---|
| Users / roles | `p01_identity` | IMPLEMENTED |
| Company / branch / fiscal / UoM | `p02_organization` | IMPLEMENTED |
| Business partner | `p04_business_partner` | IMPLEMENTED |
| Items / materials | none | PLANNED (`b07_inventory`) |
