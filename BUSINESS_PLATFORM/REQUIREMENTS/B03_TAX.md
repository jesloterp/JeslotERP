# Tax — Requirement Specification

| Field | Value |
|---|---|
| ID | `B03` |
| Package | `b03_tax` |
| Schema | `tax` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Given lines, partner tax profile, and place of supply, can the system return a deterministic tax result that later invoices must use?

Specify before sales/purchase invoices. Calculate is a service, not a print field.

## 2. In scope

- Tax code master (rate, included/excluded, recoverability, jurisdiction pointer)
- Partner tax profile on `p04` satellites (consumed, not copied)
- Pure calculate service: taxable basis, tax amount, explain trace
- Period tax register extract for posted documents
- Return / control-statement package pointer via `p23_integration`

## 3. Out of scope

- Hard-coded GST/VAT authority clients inside this module
- Finance posting of tax (finance posts what tax calculated)
- Partner legal name (p04)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Tax accountant | `tax.code.write` | Maintain codes and periods |
| System | `tax.calculate` | Called by sales/purchase/journal |

Permission codes are namespaced (`tax.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Tax code** — Effective-dated rate and indicators
- **Tax procedure / schema** — Ordered conditions — v2
- **Jurisdiction** — Pointer, not a GIS product

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `tax.calculate` | Calculation request (not a numbered doc) | `n/a` | `n/a` | None (pure) |
| `tax.register_close` | Period register close | `open → closed` | `tax.register` | Snapshot only |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Same payload + published tax version ⇒ same result.
- Explain trace lists code, basis, rate, amount.
- Invoices must persist the calculate result; they must not re-invent rates.
- Register totals must equal posted tax journals for the period.

## 8. Processes and rules

- **TAX_APPLICABILITY** — Which code applies given place of supply and partner
- **tax.return.prepare** — Optional process before filing package

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `tax.calculated`
- `tax.register.closed`
- `tax.return.exported`

### Reports (`p24_reporting` datasets — planned content)

- Tax register by code/period
- Input vs output tax

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Codes + calculate + persist on a sample journal/invoice adapter |
| Statutory | Period register = posted tax |
| Advanced | Multi-jurisdiction procedures + authority adapter |

## 14. Acceptance criteria (MVP)

- Calculate is side-effect free except optional telemetry.
- Changing a rate does not mutate historical posted invoices.
- Missing place of supply fails closed when the rule requires it.


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
