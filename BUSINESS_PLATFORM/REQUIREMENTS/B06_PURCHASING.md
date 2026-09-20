# Purchasing (Procure to Pay) — Requirement Specification

| Field | Value |
|---|---|
| ID | `B06` |
| Package | `b06_purchasing` |
| Schema | `purchasing` |
| Documentation status | **Finished specification** |
| Implementation status | `PLANNED / NOT_FOUND` |
| Recommended wave | See [README.md](README.md) |
| Product owner | Chetan Patel |

## 1. Purpose and economic question

Can a requisition become an approved PO, a goods receipt, and an AP invoice that three-way matches within tolerance?

Pair with b05 in the same commercial wave after finance + items exist.

## 2. In scope

- Purchase requisition with account assignment and process approval
- Purchase order with vendor, prices, schedule, change versions
- Goods receipt against PO (stock or consume)
- AP invoice with three-way match (PO + GR + invoice)
- Tolerance in p03 or p11 — not hardcoded

## 3. Out of scope

- Strategic sourcing / RFQ network (later)
- Warehouse putaway (b08)
- Vendor master (p04 role)

## 4. Actors and permissions

| Actor | Permission family | Responsibility |
|---|---|---|
| Requester | `purchasing.pr.write` | Create PR |
| Buyer | `purchasing.po.write` | PO |
| Receiver | `purchasing.gr.post` | GR |
| AP clerk | `purchasing.invoice.post` | AP invoice |

Permission codes are namespaced (`purchasing.*`). Identity stores them; this
module does not invent a second IAM.

## 5. Masters

- **Purchasing document type** — PR/PO/GR/IR series and match policy
- **Tolerance profile** — Qty/price/amount

Masters use `p05_metadata` layouts and `p07_number_series` when they are
numbered. They emit events; they do not post ledgers except through the
posting service of the owning module.

## 6. Transactional documents

| Document key | Name | Status model | Series object | Posts to |
|---|---|---|---|---|
| `purchasing.pr` | Requisition | `draft → approved → converted/rejected` | `purchasing.pr` | None (commitment later) |
| `purchasing.po` | Purchase order | `draft → issued → closed` | `purchasing.po` | None until GR/IR |
| `purchasing.gr` | Goods receipt | `draft → posted` | `purchasing.gr` | Stock + GR/IR |
| `purchasing.invoice` | AP invoice | `draft → matched → posted` | `purchasing.ap` | AP + GR/IR + tax |

Statuses are a documented state machine. Illegal transitions fail closed.

## 7. Posting and integrity rules

- Unapproved PR cannot convert to PO.
- Over-receipt beyond tolerance fails or starts a process.
- Match mismatch blocks post and opens inbox.
- GR and AP post through b01 and b07; idempotent on PO+line+qty/amount keys.
- PO change after issue versions the document and emits an event.

## 8. Processes and rules

- **purchasing.pr.approve** — Amount / account assignment
- **purchasing.match.exception** — Three-way fail
- **MM_TOLERANCE** — Qty/price windows

Process keys live in `p10_process`. Decision keys live in `p11_rules`.
This module starts instances and applies results; it does not embed a
second workflow engine.

## 9. Events and reports

### Events

- `purchasing.pr.approved`
- `purchasing.po.issued`
- `purchasing.gr.posted`
- `purchasing.invoice.posted`
- `purchasing.match.failed`

### Reports (`p24_reporting` datasets — planned content)

- Open PO
- GR/IR aging
- AP aging

## 13. Phased delivery

| Phase | Scope |
|---|---|
| MVP | PO + GR + AP with match; PR optional if buyer-created PO allowed by policy |
| Statutory | PR mandatory + change version + returns |
| Advanced | Scheduling agreements, subcontracting |

## 14. Acceptance criteria (MVP)

- Mismatch above tolerance cannot post AP.
- GR increases on-hand for stock items.
- AP invoice stores tax calculate result.


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
