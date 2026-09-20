# Fixed Assets — Requirement Specification

| Field | Value |
|---|---|
| ID | `B13` |
| Package | `b13_fixed_assets` |
| Schema | `assets` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can an asset be capitalized from AP or manually, then depreciated once per period through finance — idempotently?

After b01. Optional link from b06 AP.

## 2. In scope

- Asset master
- Capitalization from AP invoice or manual
- Depreciation method and useful life
- Period depreciation run via p17_scheduler
- Retirement / transfer (phase 2)

## 3. Out of scope

- Inventory items treated as assets (forbidden)
- Lease accounting (later)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Asset accountant | `assets.master.write` | Master + capitalize |
| System | `assets.depreciate.run` | Period job |

Permission codes are namespaced (`assets.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Asset class** — GL determination, default life
- **Depreciation key** — Method

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `assets.capitalization` | Capitalization | `posted` | `assets.cap` | Balance sheet |
| `assets.depreciation_run` | Depreciation run | `posted` | `assets.dep` | Expense + accum. dep. |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Capitalization posts to BS through b01.
- Second depreciation in the same period is no-op or reversal+repost by policy — never silent double expense.
- Asset number from p07.

## 8. Processes and rules

- **assets.retire.approve** — Retirement

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `assets.capitalized`
- `assets.depreciated`

### Reports (`p24_reporting` datasets — planned content)

- Asset register
- Depreciation forecast

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Manual capitalize + one method + period run |
| Statutory | From AP + retirement |
| Advanced | Parallel depreciation areas |

## 14. Acceptance criteria (MVP)

- Idempotent period run.
- Asset is not an inventory item.


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
