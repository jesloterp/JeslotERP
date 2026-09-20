# Inventory / Material Ledger (lite) — Requirement Specification

| Field | Value |
|---|---|
| ID | `B07` |
| Package | `b07_inventory` |
| Schema | `inventory` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Is on-hand stock exactly the sum of immutable movements, valued by a declared method, for item + company + location?

Second spine if finance is first. Items must exist before sales/purchase.

## 2. In scope

- Item / material master (stock, service, phantom)
- Company/item parameters (valuation method, procurable, sellable)
- Stock ledger: receipt, issue, transfer, reservation, adjustment
- Valuation: moving average and standard cost (MVP: at least one)
- Physical inventory document
- Availability = on-hand − hard reservations

## 3. Out of scope

- Bin-level WMS (b08)
- Batch/serial as a full compliance product (phase 2)
- Finance accounts (b01) — inventory posts through the port

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Master data steward | `inventory.item.write` | Item master |
| Storekeeper | `inventory.movement.post` | Movements |
| Controller | `inventory.valuation.run` | Snapshots / revalue |

Permission codes are namespaced (`inventory.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Item** — Number, type, base UoM from p02, status
- **Item company view** — Valuation method, price, flags
- **Reason code** — Adjustment reasons

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `inventory.movement` | Goods movement | `posted` | `inventory.mov` | Stock ± value + optional GL |
| `inventory.reservation` | Reservation | `active → released/expired` | `inventory.res` | Availability only |
| `inventory.count` | Physical inventory | `count → approved → posted` | `inventory.count` | Adjustment movement |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- On-hand(item, company, location) = sum(movement qty).
- Negative stock only if configuration allows.
- Posted movement lines are immutable; reverse with a new movement.
- Reservations are ledger rows, not flags. Expiry via p17_scheduler.
- Large adjustments start p10_process when rule threshold exceeded.
- Valuation snapshot period-end for reporting.

## 8. Processes and rules

- **inventory.adjust.approve** — Threshold adjustments
- **MM_NEG_STOCK** — Allow/deny negative

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `inventory.item.created`
- `inventory.movement.posted`
- `inventory.reservation.released`
- `inventory.count.posted`

### Reports (`p24_reporting` datasets — planned content)

- On-hand list
- Stock valuation dataset
- Movement register

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Item + location + receipt/issue/transfer/adjust + on-hand inquiry |
| Statutory | Valuation method + snapshot + finance post |
| Advanced | Batches, consignments, pipeline stock |

## 14. Acceptance criteria (MVP)

- After a receipt and an issue, inquiry equals arithmetic sum.
- Editing a posted movement is rejected.
- Expired reservation no longer reduces ATP.


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
