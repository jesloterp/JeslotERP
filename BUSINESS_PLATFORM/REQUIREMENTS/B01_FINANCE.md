# Finance / General Ledger — Requirement Specification

| Field | Value |
|---|---|
| ID | `B01` |
| Package | `b01_finance` |
| Schema | `finance` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Does the company know, for every posting date, what it owns, owes, earned, and spent — in a balanced, period-locked book that no logistics module can bypass?

**This is the recommended first business module.** Finish this spec's MVP in private code before starting sales or purchase.

## 2. In scope

- Chart of accounts (COA) versioned and assigned to company
- GL accounts with type, control-account flag, currency, blocked status
- Fiscal period consume from `p02_organization` (do not duplicate calendars)
- Manual journal with two-or-more lines, balanced in txn and company currency
- Post, reverse, recurring journal (via `p17_scheduler`)
- GL line inquiry and trial balance dataset
- Period open / soft-close / hard-close with audited override
- Posting interface (port) so sales, purchase, inventory, tax, assets call finance — never finance ORM
- Multi-currency amounts: transaction, company; group currency in a later phase

## 3. Out of scope

- Bank statement matching (belongs to `b04_treasury`)
- Tax calculation engine (belongs to `b03_tax`; finance stores posted tax lines)
- Cost allocations and budgets (belongs to `b02_controlling`)
- AR/AP document entry UI (owned by sales/purchase; finance owns control accounts and open-item store once those exist)
- Industry vertical packs

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Accountant | `finance.journal.write` | Draft and post journals in open periods |
| Controller | `finance.period.close` | Close periods; approve overrides |
| Auditor (read) | `finance.gl.read` | Inquiry and trial balance |
| System posting | `finance.posting.interface` | Inbound posts from other bNN |

Permission codes are namespaced (`finance.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Chart of accounts** — Named COA with version; company assignment
- **GL account** — Code, name, type (asset/liability/equity/revenue/expense), control flag, currency, blocked
- **Posting key / document type** — Journal, reverse, opening, allocation — drives number series and required fields
- **Account determination pointer** — Config/rules key for later AR/AP/GRIR — not hardcoded UUIDs

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `finance.journal` | Journal entry | `draft → posted → reversed` | `finance.journal` | GL |
| `finance.reversal` | Reversing journal | `posted` | `finance.journal` | GL (inverse) |
| `finance.recurring` | Recurring template | `active/paused` | `n/a (template)` | Creates journals |
| `finance.period_control` | Period control action | `open/soft/hard` | `n/a` | Lock only |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Reject journal if debit ≠ credit in transaction currency (and company currency after FX).
- Reject post if any account is blocked or missing from the company COA.
- Reject post if posting date falls in a hard-closed period; soft-close requires `finance.period.override` and an audit event.
- Posted journal lines are immutable. Correction = reversing document + new journal. No in-place amount edit.
- Number is allocated at first post (or at draft if legal policy requires). Void unused numbers per `p07` policy.
- Inbound posting interface is idempotent on `(source_module, source_doc, source_line, posting_key)`.
- GL inquiry totals must equal the sum of posted journal lines for the same selection.

## 8. Processes and rules

- **finance.journal.approve** — Optional approval when amount exceeds rule `FIN_JOURNAL_THRESHOLD`
- **finance.period.override** — Inbox task for posting into a soft-closed period
- **FIN_BALANCE_CHECK** — Rule: evaluate balance and currency completeness before post

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `finance.journal.posted`
- `finance.journal.reversed`
- `finance.period.soft_closed`
- `finance.period.hard_closed`
- `finance.posting.rejected`

### Reports (`p24_reporting` datasets — planned content)

- Trial balance by company / period / account (RLS on company)
- GL account statement (open item later)
- Journal register

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | COA, account, journal post/reverse, period lock, trial balance dataset, posting port stub with one manual adapter |
| Statutory | FX revaluation, opening balances, control-account open items when AR/AP exist |
| Advanced | Group currency, parallel ledgers, automated recurring at scale |

## 14. Acceptance criteria (MVP)

- Unbalanced journal returns a stable error code and does not allocate a posted number.
- Posted journal cannot be PATCH-ed on amounts; reverse creates a new numbered document.
- Hard-closed period rejects system posting from a test adapter.
- Two identical inbound posts with the same idempotency key create one journal.
- Trial balance equals journal sum for the period.
- Metadata list/form for journal and account exist in the spec names (`finance.journal`, `finance.gl_account`).


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
