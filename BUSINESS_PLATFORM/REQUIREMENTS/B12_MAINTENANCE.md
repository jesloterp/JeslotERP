# Maintenance / Plant Maintenance — Requirement Specification

| Field | Value |
|---|---|
| ID | `B12` |
| Package | `b12_maintenance` |
| Schema | `maintenance` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can equipment have planned and breakdown work orders that issue spare parts from inventory?

After inventory. Asset link is optional until b13.

## 2. In scope

- Equipment master (may link to asset in b13)
- PM work order: planned / breakdown
- Spare issue to order (b07)
- Schedule via p17 for planned dates

## 3. Out of scope

- Full EAM / reliability analytics
- Payroll time (HCM later)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Technician | `maintenance.order.confirm` | Confirm work |
| Planner | `maintenance.order.write` | Plan PM |

Permission codes are namespaced (`maintenance.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Equipment** — Functional location, serial optional
- **Task list** — Standard operations

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `maintenance.order` | Work order | `created → released → done` | `maintenance.wo` | Parts issue; optional cost object |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Parts issue is an inventory movement to the order.
- Costs settle to controlling/asset when those modules exist.

## 8. Processes and rules

- **maintenance.emergency** — Breakdown auto-start process

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `maintenance.order.released`
- `maintenance.order.completed`

### Reports (`p24_reporting` datasets — planned content)

- Backlog
- MTTR (later)

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Equipment + WO + parts issue |
| Advanced | Preventive calendar + settlement |

## 14. Acceptance criteria (MVP)

- WO cannot issue parts for a blocked item.
- Completion emits event.


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
