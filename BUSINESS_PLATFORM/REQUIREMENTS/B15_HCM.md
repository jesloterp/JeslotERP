# HCM (HR operations) — Requirement Specification

| Field | Value |
|---|---|
| ID | `B15` |
| Package | `b15_hcm` |
| Schema | `hcm` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can HR maintain employees as organizational people (already partly in p02) and later interface payroll — without making p01 the employee SoR?

Low wave. Do not block finance on HCM.

## 2. In scope

- Employee HR record distinct from login user (link by UUID)
- Assignment to org units already in p02
- Absence / attendance (phase 2)
- Payroll interface port (later; not a payroll engine in MVP)

## 3. Out of scope

- Full payroll statutory engine (FUTURE)
- Recruiting ATS (FUTURE)
- Using IAM user as the only employee master (forbidden)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| HR admin | `hcm.employee.write` | HR record |
| Manager | `hcm.team.read` | Team |

Permission codes are namespaced (`hcm.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Employee** — Person, dates, assignment
- **Action type** — Hire/change/terminate

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `hcm.action` | Personnel action | `posted` | `hcm.action` | None (or later payroll event) |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Employee ≠ user. A user may map to an employee; contractors may not have logins.
- Org assignment uses p02 departments; no duplicate org tree.

## 8. Processes and rules

- **hcm.terminate.approve** — Termination

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `hcm.employee.hired`
- `hcm.employee.terminated`

### Reports (`p24_reporting` datasets — planned content)

- Headcount by org

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Employee + link to user + org assignment |
| FUTURE | Payroll interface |

## 14. Acceptance criteria (MVP)

- Cannot delete employee with posted actions; terminate instead.
- p01 remains login SoR.


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
