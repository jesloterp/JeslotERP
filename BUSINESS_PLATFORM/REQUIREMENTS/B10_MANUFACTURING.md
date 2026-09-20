# Manufacturing — Requirement Specification

| Field | Value |
|---|---|
| ID | `B10` |
| Package | `b10_manufacturing` |
| Schema | `manufacturing` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can a production order reserve components, issue them, and receipt finished goods with a variance path into finance?

After truthful inventory and (for variance) finance.

## 2. In scope

- BOM with version and cycle detection
- Routing / operations with work-center pointer
- Production order: reserve, issue, FG receipt, status machine
- Variance posting through b01 when standard cost is used

## 3. Out of scope

- Full MES / machine integration (p23 later)
- Quality usage decision (b11)
- Detailed capacity planning (later)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Planner | `manufacturing.order.write` | Create/release order |
| Supervisor | `manufacturing.order.confirm` | Issue/receipt |

Permission codes are namespaced (`manufacturing.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **BOM** — Header, items, version, valid-from
- **Work center** — Thin master
- **Routing** — Operations sequence

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `manufacturing.order` | Production order | `created → released → in_process → delivered/closed` | `manufacturing.order` | Component issue + FG receipt + variance |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Recursive BOM rejected.
- Release creates reservations; issue posts b07 movements.
- FG receipt increases stock; variance posts through finance port.

## 8. Processes and rules

- **manufacturing.release.approve** — Optional
- **PP_COMPONENT_ATP** — Block release if ATP fail

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `manufacturing.order.released`
- `manufacturing.order.received`

### Reports (`p24_reporting` datasets — planned content)

- WIP (later)
- Order variance

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Single-level BOM + order issue/receipt |
| Advanced | Routing confirmations, WIP valuation |

## 14. Acceptance criteria (MVP)

- Cycle BOM cannot save.
- Issued qty cannot exceed reserved+policy overissue.


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
