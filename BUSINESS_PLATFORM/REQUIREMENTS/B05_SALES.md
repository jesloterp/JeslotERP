# Sales (Order to Cash) — Requirement Specification

| Field | Value |
|---|---|
| ID | `B05` |
| Package | `b05_sales` |
| Schema | `sales` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can a customer order become a delivery that issues stock and an invoice that posts AR and revenue — using partners, items, tax, and finance?

Do not implement billing as PDF-only. Depends on p04, b07 (or waiver), b01, b03.

## 2. In scope

- Quotation with validity and convert-to-order
- Sales order with ATP/reservation when item is stocked
- Outbound delivery that issues stock (`b07`)
- AR invoice from delivery/order that calls `b03` calculate and `b01` post
- Credit memo / return (phase 2)
- Print via `p32_output`

## 3. Out of scope

- CRM pipeline (b16)
- Warehouse pick tasks (b08) — delivery may post without WMS in MVP
- Pricing engine as a full condition technique (manual price in MVP)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Sales clerk | `sales.order.write` | Quote and order |
| Warehouse (optional) | `sales.delivery.post` | Post GI |
| Biller | `sales.invoice.post` | AR invoice |

Permission codes are namespaced (`sales.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Sales area / document type** — Company, currency, series
- **Pricing procedure pointer** — v2

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `sales.quote` | Quotation | `draft → released → converted/expired` | `sales.quote` | None |
| `sales.order` | Sales order | `draft → confirmed → completed/cancelled` | `sales.order` | Reservation |
| `sales.delivery` | Outbound delivery | `draft → posted` | `sales.delivery` | Stock issue + optional COGS |
| `sales.invoice` | AR invoice | `draft → posted` | `sales.invoice` | AR + revenue + tax |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Confirmed stocked order creates/releases reservation in b07.
- Delivery post is an inventory movement; cancel is a reversing movement.
- Invoice persists tax calculate result; posts through b01; open item created.
- Cannot invoice more than delivered when policy is delivery-based.
- Print is determination, not a PDF uploaded by the clerk.

## 8. Processes and rules

- **sales.credit.override** — Credit block from partner credit profile
- **sales.invoice.approve** — High-value invoice
- **SD_ATP** — Allow or deny confirm

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `sales.quote.converted`
- `sales.order.confirmed`
- `sales.delivery.posted`
- `sales.invoice.posted`

### Reports (`p24_reporting` datasets — planned content)

- Open orders
- Delivery due
- AR aging (with b01 open items)

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Order + delivery + invoice on one company, manual price, tax calculate, finance post |
| Statutory | Credit memo, returns, delivery-based billing enforcement |
| Advanced | Condition pricing, ATP across plants, intercompany |

## 14. Acceptance criteria (MVP)

- Invoice without tax calculate result is rejected when tax is in scope.
- Posting invoice twice with same key does not double AR.
- Cancel order releases reservation.
- Metadata entities named: sales.order, sales.delivery, sales.invoice.


## 10. Platform dependency map

Must consume, not reimplement: `p01_identity`, `p02_organization`,
`p03_configuration`, `p05_metadata`, `p07_number_series`, `p10_process`,
`p11_rules`, `p13_event_bus`, `p19_audit`, `p32_output`, `p33_sharing`.

Optional when the document type needs them: `p04_business_partner`,
`p08_file_media`, `p09_document`, `p14_messaging`, `p15_notification`,
`p17_scheduler`, `p24_reporting`, `p26_licensing`.

## 11. Non-claims (independent product)

- Do not copy another product's table names or document tree.
- Do not hide posting in a website controller.
- Do not ship a document that only prints.
- Do not start this module before its predecessor wave is specified.

## 12. Documentation finish checklist

- [x] Purpose and economic question written
- [x] Documents and integrity rules named
- [x] MVP acceptance criteria testable
- [ ] First module chosen by product owner (Chetan Patel)
- [ ] Implementation in private code (not this repository)
