# CRM — Requirement Specification

| Field | Value |
|---|---|
| ID | `B16` |
| Package | `b16_crm` |
| Schema | `crm` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can a lead become an opportunity against a p04 partner (or prospect) without a second party master?

After p04. Sales documents remain b05.

## 2. In scope

- Lead capture and qualify
- Opportunity with stages (process or documented state model)
- Activities (task/meeting) with due dates
- Convert to partner + optional sales quote

## 3. Out of scope

- Reviving any retired portal CRM as a second SoR (forbidden)
- Marketing automation suite (later)
- Partner golden record (p04 owns it)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Sales rep | `crm.opportunity.write` | Pipeline |
| Manager | `crm.pipeline.read` | Forecast read |

Permission codes are namespaced (`crm.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Lead source** — Catalog
- **Stage** — Process definition or enum with rules

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `crm.lead` | Lead | `new → qualified → converted/lost` | `crm.lead` | None |
| `crm.opportunity` | Opportunity | `open → won/lost` | `crm.oppty` | None until sales doc |
| `crm.activity` | Activity | `open → done` | `n/a` | None |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Convert creates or links p04 partner; does not duplicate party fields long-term.
- Won opportunity may create a sales quote; it does not post revenue.

## 8. Processes and rules

- **crm.stage.change** — Optional gated stages

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `crm.lead.converted`
- `crm.opportunity.won`
- `crm.activity.due`

### Reports (`p24_reporting` datasets — planned content)

- Pipeline by stage dataset

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | Lead + opportunity + convert to partner |
| Advanced | Activities + reporting |

## 14. Acceptance criteria (MVP)

- Convert always results in a partner id.
- No second party table.


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
