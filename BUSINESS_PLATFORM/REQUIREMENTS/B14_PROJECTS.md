# Projects — Requirement Specification

| Field | Value |
|---|---|
| ID | `B14` |
| Package | `b14_projects` |
| Schema | `projects` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can actual cost on a WBS equal assigned journals and issues, and can a milestone bill once?

After b01 and preferably b02. Billing reuses b05, does not invent AR.

## 2. In scope

- Project and WBS
- Account assignment from journal, PR/PO, issues
- Milestone or time billing creating AR via b05 interface

## 3. Out of scope

- Full PPM / resource management
- Professional services PSA suite

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Project manager | `projects.wbs.write` | Structure |
| Controller | `projects.actual.read` | Cost |
| Biller | `projects.bill.run` | Milestones |

Permission codes are namespaced (`projects.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Project** — Customer optional, company, currency
- **WBS element** — Hierarchy, billing flag

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `projects.billing` | Project billing | `draft → posted` | `projects.bill` | AR via sales/finance |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Actual cost inquiry = sum of assigned finance + inventory issues.
- Same milestone cannot bill twice.
- Cost objects may be controlling objects referenced by UUID.

## 8. Processes and rules

- **projects.milestone.approve** — Before bill

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `projects.created`
- `projects.milestone.billed`

### Reports (`p24_reporting` datasets — planned content)

- WBS actual vs plan
- Unbilled milestone

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | WBS + assignment + actual inquiry |
| Advanced | Milestone billing |

## 14. Acceptance criteria (MVP)

- Double bill of one milestone rejected.
- Inquiry matches source documents.


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
