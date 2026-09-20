# Service — Requirement Specification

| Field | Value |
|---|---|
| ID | `B17` |
| Package | `b17_service` |
| Schema | `service` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can a service order against a partner (and optional equipment) keep an SLA clock and issue parts from inventory?

After CRM or p04, inventory, and preferably maintenance equipment.

## 2. In scope

- Service order
- SLA clocks via process/scheduler (not UI-only)
- Parts issue (b07)
- Optional billing via b05

## 3. Out of scope

- Field-service geo dispatch product
- ITSM enterprise suite

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Agent | `service.order.write` | Create/update |
| Technician | `service.order.confirm` | Confirm + parts |

Permission codes are namespaced (`service.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Service product / catalog** — Thin
- **SLA profile** — Response/resolve times

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `service.order` | Service order | `open → in_progress → resolved/closed` | `service.order` | Parts; optional AR |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- SLA breach creates escalation task.
- Parts issue is inventory movement.
- Billing reuses sales/finance — no shadow AR.

## 8. Processes and rules

- **service.sla.breach** — Escalation
- **service.order.approve** — Warranty / chargeable

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `service.order.opened`
- `service.sla.breached`
- `service.order.closed`

### Reports (`p24_reporting` datasets — planned content)

- Open orders
- SLA breach list

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Order + SLA clock + parts |
| Advanced | Contracts + billing |

## 14. Acceptance criteria (MVP)

- SLA timer exists without opening the UI.
- Breach creates a process task.


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
