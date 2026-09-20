# Warehouse — Requirement Specification

| Field | Value |
|---|---|
| ID | `B08` |
| Package | `b08_warehouse` |
| Schema | `warehouse` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can a receipt be put away into bins and a delivery be picked without changing the inventory valuation point unless explicitly designed?

After b07. Do not make bins the GL valuation point in MVP.

## 2. In scope

- Bin master under a warehouse location
- Putaway task from GR
- Pick / pack task for outbound delivery
- Bin qty that reconciles to inventory location (MVP: warehouse not valuation-relevant)

## 3. Out of scope

- Valuation (b07)
- Carrier execution (b09)
- Automation / PLC

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Warehouse clerk | `warehouse.task.confirm` | Confirm putaway/pick |
| Supervisor | `warehouse.bin.write` | Bin master |

Permission codes are namespaced (`warehouse.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Bin** — Coordinate, type, capacity, warehouse
- **Strategy** — Putaway/pick strategy key

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `warehouse.putaway` | Putaway task | `open → confirmed` | `warehouse.task` | Bin qty |
| `warehouse.pick` | Pick task | `open → confirmed/short` | `warehouse.task` | Bin qty; short-pick process |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Confirm putaway only if strategy and capacity pass.
- Short-pick starts a process; delivery quantity may reduce by policy.
- Bin sum for a location equals inventory on-hand when WMS is active.

## 8. Processes and rules

- **warehouse.short_pick** — Exception inbox

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `warehouse.task.confirmed`
- `warehouse.pick.short`

### Reports (`p24_reporting` datasets — planned content)

- Open tasks
- Bin occupancy

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Bins + confirm putaway/pick against existing GR/delivery |
| Advanced | Waves, valuation-relevant warehouse |

## 14. Acceptance criteria (MVP)

- Pick cannot confirm more than bin qty.
- Inventory location qty still matches b07 after a full putaway+pick cycle in the test script.


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
