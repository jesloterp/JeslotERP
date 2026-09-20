# Treasury / Cash and Banks — Requirement Specification

| Field | Value |
|---|---|
| ID | `B04` |
| Package | `b04_treasury` |
| Schema | `treasury` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can the company propose, approve, and post payments against open items using house banks — without storing PSP credentials in this module?

Requires b01 and open items (from AR/AP). Do not start as a standalone bank app.

## 2. In scope

- House bank and account master
- Payment method catalog
- Payment proposal from AR/AP open items
- Payment document that clears open items via finance
- Bank statement import (file via p08) and matching
- PSP orchestration through `p23_integration` / licensing ports — secret refs only

## 3. Out of scope

- GL of cash accounts (b01 owns accounts)
- Partner bank satellites (p04)
- Dunning content (notification templates in p15; trigger here or AR)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Treasurer | `treasury.payment.propose` | Build proposals |
| Approver | `treasury.payment.approve` | Release payments |
| Cashier | `treasury.statement.import` | Import statements |

Permission codes are namespaced (`treasury.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **House bank** — Bank, account, company, GL cash account pointer
- **Payment method** — Instrument, clearing days, series

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `treasury.proposal` | Payment proposal | `draft → approved → paid` | `treasury.proposal` | None until pay |
| `treasury.payment` | Payment document | `posted` | `treasury.payment` | Bank + AP/AR + clearing |
| `treasury.statement` | Bank statement | `imported → matched` | `treasury.stmt` | Optional residual |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Payment posts through b01; cash and clearing accounts from determination.
- Cannot pay a hard-closed period.
- Proposal approval is `p10_process` above a rule threshold.
- No raw PSP keys in treasury tables.

## 8. Processes and rules

- **treasury.payment.release** — Dual control above threshold
- **TRE_PAY_LIMIT** — Rule: amount vs role

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `treasury.proposal.approved`
- `treasury.payment.posted`
- `treasury.statement.matched`

### Reports (`p24_reporting` datasets — planned content)

- Payment register
- Unmatched statement items
- Cash position (later)

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | House bank + manual payment against an open item |
| Statutory | Proposal + bank statement matching |
| Advanced | PSP adapter + cash forecast |

## 14. Acceptance criteria (MVP)

- Payment reduces open item residual to documented amount.
- Duplicate payment idempotency key does not double-clear.
- Statement import stores file in p08 and never inline secrets.


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
