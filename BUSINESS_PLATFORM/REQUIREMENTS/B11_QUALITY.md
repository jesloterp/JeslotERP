# Quality — Requirement Specification

| Field | Value |
|---|---|
| ID | `B11` |
| Package | `b11_quality` |
| Schema | `quality` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can an inspection lot block usage of received or produced stock until a usage decision?

After b07 and optionally b10.

## 2. In scope

- Inspection lot from GR or production receipt
- Results recording (qualitative/quantitative)
- Usage decision: accept / reject / rework
- NCR (non-conformance) with process

## 3. Out of scope

- LIMS instruments
- Legal device manufacturing (industry pack)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Inspector | `quality.lot.record` | Results |
| QA lead | `quality.ud.decide` | Usage decision |

Permission codes are namespaced (`quality.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Inspection plan** — Characteristics per item
- **NCR type** — Category

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `quality.lot` | Inspection lot | `created → recorded → decided` | `quality.lot` | Stock status / blocked stock |
| `quality.ncr` | NCR | `open → closed` | `quality.ncr` | Optional scrap movement |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Stock from GR can be inspection-held; ATP excludes it until UD.
- Reject UD may trigger return or scrap movement in b07.

## 8. Processes and rules

- **quality.ncr.approve** — Disposition

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `quality.lot.created`
- `quality.ud.decided`
- `quality.ncr.opened`

### Reports (`p24_reporting` datasets — planned content)

- Open lots
- Reject rate

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Lot + UD + block ATP |
| Advanced | Plans, SPC |

## 14. Acceptance criteria (MVP)

- ATP does not include inspection-held qty.
- UD is audited.


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
