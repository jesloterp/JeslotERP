# Business Platform TODO

This list is planning only. It does **not** authorize source files, SQL, or
APIs inside `public-platform-docs/`.

Status values use the platform vocabulary. Almost every item is `TODO` or
`PLANNED` because **no business package implementation was found**.

**Item count:** 52

## Foundation

### BIZ-FND-001 — Choose first business module

| Field | Value |
|---|---|
| ID | `BIZ-FND-001` |
| Module | Foundation |
| Feature | Choose first business module |
| Description | Record an explicit product decision for the first `bNN` so implementation does not fork across seventeen empty packages. |
| Dependencies | Kernel SoR-Live; stakeholder decision |
| Priority | Critical |
| Status | `TODO` |
| Implementation notes | Do not start BIZ-FND-002 for more than one module until this is decided. |
| Acceptance criteria | A written choice of b01, b05, b07, or another registered module exists; registry updated. |

### BIZ-FND-002 — Business module plugin skeleton

| Field | Value |
|---|---|
| ID | `BIZ-FND-002` |
| Module | Foundation |
| Feature | Business module plugin skeleton |
| Description | Create `business/bNN_*` with ModulePlugin, permission catalog, empty schema module, and docs GUIDE/SCHEMA/API — no fake ledgers. |
| Dependencies | BIZ-FND-001; p01 permission seed pattern |
| Priority | Critical |
| Status | `TODO` |
| Implementation notes | Mirror platform plugin loading; do not mount placeholder CRUD that pretends to post. |
| Acceptance criteria | Module loads in dependency order; health lists it; tests prove mount; no business tables claimed as live without tests. |

### BIZ-FND-003 — Spine wiring checklist

| Field | Value |
|---|---|
| ID | `BIZ-FND-003` |
| Module | Foundation |
| Feature | Spine wiring checklist |
| Description | Define the mandatory kernel adapters every business document must call: number allocate, metadata entity, process start, audit emit, share evaluate, output determine, entitlement module code. |
| Dependencies | p07, p05, p10, p19, p33, p32, p26 |
| Priority | Critical |
| Status | `TODO` |
| Implementation notes | A reusable application port bundle — not copy-paste per module. |
| Acceptance criteria | Checklist published; first document type implements every row or records an explicit waiver. |

### BIZ-FND-004 — Metadata entity seed for primary document

| Field | Value |
|---|---|
| ID | `BIZ-FND-004` |
| Module | Foundation |
| Feature | Metadata entity seed for primary document |
| Description | Seed dictionary + list/form/filter/action/inspector layouts for the first business document. |
| Dependencies | p05_metadata; BIZ-FND-001 |
| Priority | High |
| Status | `TODO` |
| Implementation notes | Follow bp.partner / org.company layout contract; do not invent a new UI stack. |
| Acceptance criteria | Published UI pack contains the entity; validate API enforces required_if; list sort allowlist documented. |

### BIZ-FND-005 — Period and posting calendar contract

| Field | Value |
|---|---|
| ID | `BIZ-FND-005` |
| Module | Foundation |
| Feature | Period and posting calendar contract |
| Description | Agree how fiscal periods from `p02_organization` lock business postings. |
| Dependencies | p02_organization fiscal; b01 when present |
| Priority | High |
| Status | `TODO` |
| Implementation notes | Even inventory-first builds need a posting date rule. |
| Acceptance criteria | Attempting to post into a closed period returns a stable business error code. |

## Master Data

### BIZ-MST-001 — Item / material master

| Field | Value |
|---|---|
| ID | `BIZ-MST-001` |
| Module | Master Data |
| Feature | Item / material master |
| Description | Implement item master with identifiers, descriptions, base UoM, item type (stock / service / phantom), status, and company-level views. |
| Dependencies | p02 UoM; p05; p07; p04 for manufacturer/vendor refs |
| Priority | Critical |
| Status | `PLANNED` |
| Implementation notes | Lives in b07_inventory, not a new platform. |
| Acceptance criteria | Create/update/deactivate item; unique number; metadata list/form; audit event. |

### BIZ-MST-002 — Item plant / company parameters

| Field | Value |
|---|---|
| ID | `BIZ-MST-002` |
| Module | Master Data |
| Feature | Item plant / company parameters |
| Description | Store per-company procurement, sales, and valuation-relevant parameters without duplicating the global item. |
| Dependencies | BIZ-MST-001; p02 company |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | No cross-schema FK to org; UUID + gateway. |
| Acceptance criteria | Same item can be stocked in company A and blocked in company B. |

### BIZ-MST-003 — Partner role completeness for ERP

| Field | Value |
|---|---|
| ID | `BIZ-MST-003` |
| Module | Master Data |
| Feature | Partner role completeness for ERP |
| Description | Ensure customer and vendor roles on `p04_business_partner` carry the attributes sales/purchase posting need (payment terms pointer, reconciliation account hint). |
| Dependencies | p04_business_partner; b01 account determination later |
| Priority | High |
| Status | `PARTIALLY_IMPLEMENTED` |
| Implementation notes | Partner kernel exists; ERP-specific attributes must be additive, not a fork. |
| Acceptance criteria | Customer/vendor role can be assigned; missing posting attributes fail closed when finance goes live. |

### BIZ-MST-004 — Bank and payment-method masters

| Field | Value |
|---|---|
| ID | `BIZ-MST-004` |
| Module | Master Data |
| Feature | Bank and payment-method masters |
| Description | House-bank and payment-method catalog owned by treasury/finance, referencing partner bank satellites. |
| Dependencies | p04 bank satellites; b04 / b01 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Do not store raw secrets; use p03 refs if credentials appear. |
| Acceptance criteria | House bank can be selected on a payment proposal. |

### BIZ-MST-005 — Tax code master

| Field | Value |
|---|---|
| ID | `BIZ-MST-005` |
| Module | Master Data |
| Feature | Tax code master |
| Description | Tax codes with rate, included/excluded, recoverability, and jurisdiction pointer — calculation engine may come later. |
| Dependencies | b03_tax; b01 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Codes are master data; filing is a later process. |
| Acceptance criteria | Tax code CRUD with effective dating; used by sales/purchase line preview once those exist. |

## Finance

### BIZ-FIN-001 — Chart of accounts and GL accounts

| Field | Value |
|---|---|
| ID | `BIZ-FIN-001` |
| Module | Finance |
| Feature | Chart of accounts and GL accounts |
| Description | Implement a company-assignable chart of accounts with account type, control-account flags, currency, and blocked-for-posting status. |
| Dependencies | p02 company; p05; p07 optional account numbers |
| Priority | Critical |
| Status | `PLANNED` |
| Implementation notes | First object in b01_finance. |
| Acceptance criteria | COA version published; accounts listed via metadata; cannot post to blocked account. |

### BIZ-FIN-002 — Journal entry with balanced lines

| Field | Value |
|---|---|
| ID | `BIZ-FIN-002` |
| Module | Finance |
| Feature | Journal entry with balanced lines |
| Description | Create a journal document that requires at least two lines, balanced in transaction and company currency, with period derivation from posting date. |
| Dependencies | BIZ-FIN-001; p02 fiscal; p07; p10 optional approve; p19 |
| Priority | Critical |
| Status | `PLANNED` |
| Implementation notes | No inventory or tax required for a manual journal. |
| Acceptance criteria | Unbalanced journal rejected; posted journal immutable except reversing document; number allocated once. |

### BIZ-FIN-003 — General-ledger aggregate and line inquiry

| Field | Value |
|---|---|
| ID | `BIZ-FIN-003` |
| Module | Finance |
| Feature | General-ledger aggregate and line inquiry |
| Description | Maintain GL open-item or balance inquiry by account, company, period, and currency without scanning journals ad hoc. |
| Dependencies | BIZ-FIN-002 |
| Priority | Critical |
| Status | `PLANNED` |
| Implementation notes | Inquiry is not a spreadsheet export only. |
| Acceptance criteria | Posted journal appears on account statement; totals match journal sum. |

### BIZ-FIN-004 — Period open / close and hard close

| Field | Value |
|---|---|
| ID | `BIZ-FIN-004` |
| Module | Finance |
| Feature | Period open / close and hard close |
| Description | Allow controllers to open, soft-close, and hard-close periods with override permission and audit. |
| Dependencies | p02 fiscal; p01 permissions; p19 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Hard close is irreversible without a compensating period. |
| Acceptance criteria | Posting to hard-closed period denied; override is permissioned and audited. |

### BIZ-FIN-005 — Reversing and recurring journals

| Field | Value |
|---|---|
| ID | `BIZ-FIN-005` |
| Module | Finance |
| Feature | Reversing and recurring journals |
| Description | Support full reversal documents and scheduled recurring templates via p17_scheduler. |
| Dependencies | BIZ-FIN-002; p17_scheduler |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Recurring is a schedule + template, not cron in the API host. |
| Acceptance criteria | Reversal balances original; recurring run creates a new numbered journal. |

### BIZ-FIN-006 — Multi-currency valuation

| Field | Value |
|---|---|
| ID | `BIZ-FIN-006` |
| Module | Finance |
| Feature | Multi-currency valuation |
| Description | Store transaction, company, and optional group currency; revalue open items using p02 FX. |
| Dependencies | p02 FX; BIZ-FIN-003 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Do not invent a second FX table. |
| Acceptance criteria | Revaluation posts difference journals; rates resolved from organization FX. |

## Accounting

### BIZ-ACC-001 — AR / AP control-account determination

| Field | Value |
|---|---|
| ID | `BIZ-ACC-001` |
| Module | Accounting |
| Feature | AR / AP control-account determination |
| Description | Determine reconciliation accounts from partner role + company + document type via rules or configuration, not hardcoded ids. |
| Dependencies | p04; b01; p11_rules recommended; p03 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Determination is data. |
| Acceptance criteria | Missing determination fails closed with a stable code. |

### BIZ-ACC-002 — Subledger open items

| Field | Value |
|---|---|
| ID | `BIZ-ACC-002` |
| Module | Accounting |
| Feature | Subledger open items |
| Description | Maintain partner open items for invoices, credit notes, and payments with residual clearing. |
| Dependencies | BIZ-FIN-002; p04; b05/b06 later |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | GL control account must equal subledger sum. |
| Acceptance criteria | Clearing document reduces open items; reconciliation report matches. |

### BIZ-ACC-003 — Automatic posting from logistics documents

| Field | Value |
|---|---|
| ID | `BIZ-ACC-003` |
| Module | Accounting |
| Feature | Automatic posting from logistics documents |
| Description | Define posting interfaces so GR/IR, COGS, and revenue documents create journals without logistics importing finance ORM. |
| Dependencies | b07, b05, b06, b01 events |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Event + posting service. |
| Acceptance criteria | Goods receipt posts GR/IR; reversal posts inverse; idempotent on business key. |

## Sales

### BIZ-SAL-001 — Sales quotation

| Field | Value |
|---|---|
| ID | `BIZ-SAL-001` |
| Module | Sales |
| Feature | Sales quotation |
| Description | Quotation document with partner, items, quantities, prices, validity, and convert-to-order action. |
| Dependencies | p04; BIZ-MST-001; p07; p05; p10 optional |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Prices may be manual in v1; pricing engine later. |
| Acceptance criteria | Quote numbered; convert copies lines to an order with reference. |

### BIZ-SAL-002 — Sales order with ATP check

| Field | Value |
|---|---|
| ID | `BIZ-SAL-002` |
| Module | Sales |
| Feature | Sales order with ATP check |
| Description | Sales order that reserves or checks available stock before confirm when the item is stocked. |
| Dependencies | BIZ-SAL-001; b07 reservation |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Fail closed if ATP denied and policy requires it. |
| Acceptance criteria | Confirmed order creates reservation; cancel releases it. |

### BIZ-SAL-003 — Outbound delivery

| Field | Value |
|---|---|
| ID | `BIZ-SAL-003` |
| Module | Sales |
| Feature | Outbound delivery |
| Description | Delivery document that issues stock and is referenceable by billing. |
| Dependencies | BIZ-SAL-002; b07 issue; p10 pick optional |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Warehouse pick may come later; inventory issue cannot. |
| Acceptance criteria | Posting delivery reduces unrestricted stock and writes a movement. |

### BIZ-SAL-004 — AR billing document

| Field | Value |
|---|---|
| ID | `BIZ-SAL-004` |
| Module | Sales |
| Feature | AR billing document |
| Description | Invoice from deliveries or orders that posts AR and revenue via finance posting interface. |
| Dependencies | BIZ-SAL-003; b01; b03 tax; p07; p32 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Print via p32_output; number via p07. |
| Acceptance criteria | Invoice numbered, taxed, posted, printable; duplicate submit idempotent. |

## Purchase

### BIZ-PUR-001 — Purchase requisition

| Field | Value |
|---|---|
| ID | `BIZ-PUR-001` |
| Module | Purchase |
| Feature | Purchase requisition |
| Description | Internal request for items/services with requester, account assignment, and approve via p10_process. |
| Dependencies | BIZ-MST-001; p10; p01; p07 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Approval is process, not a boolean. |
| Acceptance criteria | Unapproved PR cannot convert to PO. |

### BIZ-PUR-002 — Purchase order

| Field | Value |
|---|---|
| ID | `BIZ-PUR-002` |
| Module | Purchase |
| Feature | Purchase order |
| Description | PO with vendor, items, prices, delivery schedule, and change-history. |
| Dependencies | p04 vendor; BIZ-PUR-001; p07; p19 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Changes after transmit should version. |
| Acceptance criteria | PO numbered; printable; change event emitted. |

### BIZ-PUR-003 — Goods receipt against PO

| Field | Value |
|---|---|
| ID | `BIZ-PUR-003` |
| Module | Purchase |
| Feature | Goods receipt against PO |
| Description | GR that increases stock or consumes to account assignment and posts GR/IR. |
| Dependencies | BIZ-PUR-002; b07 receipt; b01 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Over-receipt policy is configuration. |
| Acceptance criteria | GR movement and journal are idempotent on PO+line+qty key. |

### BIZ-PUR-004 — AP invoice and three-way match

| Field | Value |
|---|---|
| ID | `BIZ-PUR-004` |
| Module | Purchase |
| Feature | AP invoice and three-way match |
| Description | Vendor invoice matched to PO and GR within tolerance; exceptions start a process. |
| Dependencies | BIZ-PUR-003; b01; b03; p10; p11 tolerances |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Tolerance in p03 or p11, not hardcoded. |
| Acceptance criteria | Mismatch blocks post and opens an inbox task. |

## Inventory

### BIZ-INV-001 — Stock ledger

| Field | Value |
|---|---|
| ID | `BIZ-INV-001` |
| Module | Inventory |
| Feature | Stock ledger |
| Description | Implement a stock ledger supporting receipt, issue, transfer, reservation, adjustment, valuation snapshot, and audit history per item, company, and location. |
| Dependencies | BIZ-MST-001; p02 warehouse/location; p19; p07 for adjustment docs |
| Priority | Critical |
| Status | `PLANNED` |
| Implementation notes | This is the heart of b07. Do not ship item master without movements. |
| Acceptance criteria | Each movement is an immutable line; on-hand equals sum of movements; negative stock only if policy allows. |

### BIZ-INV-002 — Valuation methods

| Field | Value |
|---|---|
| ID | `BIZ-INV-002` |
| Module | Inventory |
| Feature | Valuation methods |
| Description | Support at least moving-average and standard-cost valuation, with period snapshots for reporting. |
| Dependencies | BIZ-INV-001; b01 for revaluation journals |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Method is per company/item. |
| Acceptance criteria | Receipt reprices moving average; standard-cost difference posts a variance account when finance exists. |

### BIZ-INV-003 — Reservation and availability

| Field | Value |
|---|---|
| ID | `BIZ-INV-003` |
| Module | Inventory |
| Feature | Reservation and availability |
| Description | Hard and soft reservations with expiry, used by sales and production. |
| Dependencies | BIZ-INV-001; b05; b10 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Reservations are ledger rows, not flags. |
| Acceptance criteria | Available = on-hand − hard reservations; expired reservation auto-releases via scheduler. |

### BIZ-INV-004 — Physical inventory / adjustment document

| Field | Value |
|---|---|
| ID | `BIZ-INV-004` |
| Module | Inventory |
| Feature | Physical inventory / adjustment document |
| Description | Counted quantity produces an adjustment movement with reason code and approval threshold. |
| Dependencies | BIZ-INV-001; p10; p11 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Large adjustments require process. |
| Acceptance criteria | Count document numbered; posts only after approval when threshold exceeded. |

## Warehouse

### BIZ-WH-001 — Bin master and putaway

| Field | Value |
|---|---|
| ID | `BIZ-WH-001` |
| Module | Warehouse |
| Feature | Bin master and putaway |
| Description | Bins under a warehouse location with putaway that splits an inventory receipt into bin quantities. |
| Dependencies | b07; p02 warehouse |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Inventory location remains the valuation point unless warehouse is valuation-relevant (v2). |
| Acceptance criteria | Putaway task completes only if bin capacities and item strategies pass. |

### BIZ-WH-002 — Pick / pack / ship tasks

| Field | Value |
|---|---|
| ID | `BIZ-WH-002` |
| Module | Warehouse |
| Feature | Pick / pack / ship tasks |
| Description | Wave or order-based pick tasks that confirm before outbound delivery post. |
| Dependencies | BIZ-WH-001; b05 delivery |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Can start after inventory issue exists. |
| Acceptance criteria | Pick confirmation reduces bin qty; short-pick starts an exception process. |

## Manufacturing

### BIZ-MFG-001 — BOM and routing

| Field | Value |
|---|---|
| ID | `BIZ-MFG-001` |
| Module | Manufacturing |
| Feature | BOM and routing |
| Description | Bill of materials with versioning and routing operations with work-center pointers. |
| Dependencies | BIZ-MST-001; b02 cost objects later |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | BOM is master data, not an Excel import only. |
| Acceptance criteria | BOM version effective-dated; cycle detection rejects recursive BOMs. |

### BIZ-MFG-002 — Production order with issue and receipt

| Field | Value |
|---|---|
| ID | `BIZ-MFG-002` |
| Module | Manufacturing |
| Feature | Production order with issue and receipt |
| Description | Order that reserves components, issues to order, and receipts finished goods with variance. |
| Dependencies | BIZ-MFG-001; b07; b01 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Variance posting needs finance. |
| Acceptance criteria | Order status machine; component issue and FG receipt are stock movements. |

## CRM

### BIZ-CRM-001 — Lead and opportunity

| Field | Value |
|---|---|
| ID | `BIZ-CRM-001` |
| Module | CRM |
| Feature | Lead and opportunity |
| Description | Lead capture, qualify, convert to opportunity against p04 partner or prospect, with stage process. |
| Dependencies | p04; p10; p15 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Do not revive retired portal CRM leftovers as a second SoR. |
| Acceptance criteria | Convert creates or links a partner; stage changes are process or documented state model. |

### BIZ-CRM-002 — Activity and pipeline reporting

| Field | Value |
|---|---|
| ID | `BIZ-CRM-002` |
| Module | CRM |
| Feature | Activity and pipeline reporting |
| Description | Tasks, meetings, and a dataset on p24_reporting for pipeline by stage. |
| Dependencies | BIZ-CRM-001; p17; p24 |
| Priority | Low |
| Status | `PLANNED` |
| Implementation notes | Reporting is definitions, not a private SQL page. |
| Acceptance criteria | Activities due appear in process inbox or a CRM inbox; dataset publishes. |

## Logistics

### BIZ-LOG-001 — Consignment / shipment document

| Field | Value |
|---|---|
| ID | `BIZ-LOG-001` |
| Module | Logistics |
| Feature | Consignment / shipment document |
| Description | Transport document linking deliveries or independent consignments with origin, destination, carrier partner, and freight terms. |
| Dependencies | p04; b05/b06; p07; p09 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Industry-specific fields belong in overlays later, not kernel. |
| Acceptance criteria | Document numbered; links to source deliveries; status events emitted. |

### BIZ-LOG-002 — Trip / resource assignment

| Field | Value |
|---|---|
| ID | `BIZ-LOG-002` |
| Module | Logistics |
| Feature | Trip / resource assignment |
| Description | Assign vehicle/driver or carrier resources and capture actual departure/arrival. |
| Dependencies | BIZ-LOG-001; p02 locations |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Fleet master may start thin. |
| Acceptance criteria | Trip cannot complete without assigned resource when policy requires it. |

### BIZ-LOG-003 — Freight costing interface

| Field | Value |
|---|---|
| ID | `BIZ-LOG-003` |
| Module | Logistics |
| Feature | Freight costing interface |
| Description | Estimate and actual freight that can post to finance and optionally invoice a customer. |
| Dependencies | BIZ-LOG-001; b01; b05 |
| Priority | Low |
| Status | `PLANNED` |
| Implementation notes | Wait for finance posting interface. |
| Acceptance criteria | Freight accrual journal is idempotent per shipment. |

## Tax

### BIZ-TAX-001 — Tax calculation service

| Field | Value |
|---|---|
| ID | `BIZ-TAX-001` |
| Module | Tax |
| Feature | Tax calculation service |
| Description | Given lines, partner tax profile, place of supply, and codes, return taxable basis, tax amount, and recoverability without posting. |
| Dependencies | BIZ-MST-005; p04 tax satellites; p02 tax registrations |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Pure function + versioned rules; p11 may host predicates. |
| Acceptance criteria | Same payload always same result for a published tax version; explain trace available. |

### BIZ-TAX-002 — Return / control-statement coordination

| Field | Value |
|---|---|
| ID | `BIZ-TAX-002` |
| Module | Tax |
| Feature | Return / control-statement coordination |
| Description | Period tax register extract and filing package pointer via p23_integration — not a hard-coded authority client. |
| Dependencies | BIZ-TAX-001; b01; p23; p24 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Authority connectors are integration adapters. |
| Acceptance criteria | Register totals equal posted tax journals for the period. |

## Projects

### BIZ-PRJ-001 — WBS and project costing

| Field | Value |
|---|---|
| ID | `BIZ-PRJ-001` |
| Module | Projects |
| Feature | WBS and project costing |
| Description | Project with WBS elements that can be account-assigned from sales, purchase, and journals. |
| Dependencies | b01; b02; b05 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Cost objects may live in controlling and be referenced here. |
| Acceptance criteria | Actual cost inquiry equals sum of assigned journals and issues. |

### BIZ-PRJ-002 — Project billing

| Field | Value |
|---|---|
| ID | `BIZ-PRJ-002` |
| Module | Projects |
| Feature | Project billing |
| Description | Milestone or time-based billing creating AR invoices. |
| Dependencies | BIZ-PRJ-001; b05 billing |
| Priority | Low |
| Status | `PLANNED` |
| Implementation notes | Reuse sales billing interface. |
| Acceptance criteria | Milestone complete → invoice draft; no double bill on the same milestone. |

## Service

### BIZ-SRV-001 — Service order and SLA

| Field | Value |
|---|---|
| ID | `BIZ-SRV-001` |
| Module | Service |
| Feature | Service order and SLA |
| Description | Service order against a partner and optional asset/equipment with SLA clocks from process/scheduler. |
| Dependencies | p04; b12 or asset; p10; p17 |
| Priority | Low |
| Status | `PLANNED` |
| Implementation notes | SLA timers must not be UI-only. |
| Acceptance criteria | Breach creates escalation task; parts issue writes inventory movement. |

## Assets

### BIZ-AST-001 — Asset master and capitalization

| Field | Value |
|---|---|
| ID | `BIZ-AST-001` |
| Module | Assets |
| Feature | Asset master and capitalization |
| Description | Asset master created from AP invoice or manual capitalization, linked to a GL asset account. |
| Dependencies | b01; b06 optional |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Do not treat assets as inventory items. |
| Acceptance criteria | Capitalization posts to balance sheet; asset number allocated. |

### BIZ-AST-002 — Depreciation run

| Field | Value |
|---|---|
| ID | `BIZ-AST-002` |
| Module | Assets |
| Feature | Depreciation run |
| Description | Period depreciation by method and useful life, scheduled through p17_scheduler, posting through finance. |
| Dependencies | BIZ-AST-001; p17; b01 |
| Priority | Medium |
| Status | `PLANNED` |
| Implementation notes | Idempotent per asset+period. |
| Acceptance criteria | Second run in the same period is a no-op or reversal+repost by policy. |

## Reporting

### BIZ-RPT-001 — Trial balance dataset

| Field | Value |
|---|---|
| ID | `BIZ-RPT-001` |
| Module | Reporting |
| Feature | Trial balance dataset |
| Description | Publish a p24_reporting dataset for trial balance by company and period with RLS. |
| Dependencies | b01; p24; p05; p33/p02 RLS |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | No private SQL console. |
| Acceptance criteria | Dataset totals equal GL; unauthorized company is empty/fail-closed. |

### BIZ-RPT-002 — Stock valuation dataset

| Field | Value |
|---|---|
| ID | `BIZ-RPT-002` |
| Module | Reporting |
| Feature | Stock valuation dataset |
| Description | On-hand and value by item/location as of period end. |
| Dependencies | b07; p24 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | Snapshot table recommended. |
| Acceptance criteria | Report matches ledger valuation within documented rounding. |

### BIZ-RPT-003 — AR / AP aging dataset

| Field | Value |
|---|---|
| ID | `BIZ-RPT-003` |
| Module | Reporting |
| Feature | AR / AP aging dataset |
| Description | Open-item aging buckets for partners. |
| Dependencies | BIZ-ACC-002; p24 |
| Priority | High |
| Status | `PLANNED` |
| Implementation notes | As-of date parameter required. |
| Acceptance criteria | Aging sum equals subledger open items. |

## Industry Extensions

### BIZ-IND-001 — Pack format for vertical fields

| Field | Value |
|---|---|
| ID | `BIZ-IND-001` |
| Module | Industry |
| Feature | Pack format for vertical fields |
| Description | Define how industry fields, layouts, and rules travel as p30_alm + p05 packages without new pNN numbers. |
| Dependencies | p05; p11; p30; p06 |
| Priority | Low |
| Status | `FUTURE` |
| Implementation notes | Registry forbids verticals in b01–b17. |
| Acceptance criteria | A sample pack installs overlays and uninstalls cleanly on a disposable tenant. |

### BIZ-IND-002 — Reserve b18+ only when a vertical is approved

| Field | Value |
|---|---|
| ID | `BIZ-IND-002` |
| Module | Industry |
| Feature | Reserve b18+ only when a vertical is approved |
| Description | Do not invent travel, healthcare, or other verticals in the horizontal registry. |
| Dependencies | Business registry governance |
| Priority | Low |
| Status | `FUTURE` |
| Implementation notes | Name the vertical when it exists; do not squat numbers. |
| Acceptance criteria | Registry updated only after written approval. |
