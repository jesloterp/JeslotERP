# Controlling / Management Accounting — Requirement Specification

| Field | Value |
|---|---|
| ID | `B02` |
| Package | `b02_controlling` |
| Schema | `controlling` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can management see cost and budget by cost object without a second set of books that disagrees with `b01_finance`?

Depends on `b01_finance` MVP. Do not build a second GL.

## 2. In scope

- Cost centers, profit centers, internal orders (cost objects)
- Budget version by object and period
- Allocation cycles (assessment / distribution) that post to finance via the posting port
- Actuals inquiry = finance lines tagged with cost object

## 3. Out of scope

- Legal GL (b01)
- Payroll costing (later HCM interface)
- Project WBS (b14 may reference controlling objects)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Controller | `controlling.object.write` | Maintain cost objects and budgets |
| Manager (read) | `controlling.actual.read` | Own-object actuals |

Permission codes are namespaced (`controlling.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Cost object** — Type, owner, company, validity
- **Budget version** — Period, amount, currency, status
- **Allocation cycle** — Sender, receiver, rule, next-run

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `controlling.allocation_run` | Allocation run | `draft → posted` | `controlling.alloc` | Finance journals |
| `controlling.budget_release` | Budget release | `draft → released` | `n/a` | None (control) |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Every allocation journal is balanced and tagged with sender/receiver objects.
- Budget check is a rule (`CO_BUDGET_HARD`) invoked by purchase/project — fail closed if hard.
- Actuals are not a separately typed ledger; they are finance lines with object dimensions.

## 8. Processes and rules

- **controlling.alloc.approve** — Approve allocation cycle before run
- **CO_BUDGET_HARD** — Block commitment when remaining budget < request

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `controlling.allocation.posted`
- `controlling.budget.released`
- `controlling.budget.exceeded`

### Reports (`p24_reporting` datasets — planned content)

- Cost center actual vs budget
- Allocation run log

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Cost center master + dimension on finance lines + budget store without hard stop |
| Statutory | Hard budget on PR/PO |
| Advanced | Cycles, profit center, internal orders |

## 14. Acceptance criteria (MVP)

- Finance journal can carry a cost center; inquiry filters by it.
- Allocation run posts through b01 port and is idempotent per cycle+period.
- Missing cost object on a required document type fails validate.


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
