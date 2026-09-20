# How to Plan a Business Module

This is the **mandatory planning method** for every `bNN_*` module. It exists
so JeslotERP does not become another form-first ERP. Chetan Patel’s product
goal is **enterprise posting discipline**: documents post, periods lock,
stock balances, and the kernel is reused.

**Status of all business modules today:** `PLANNED` / `NOT_FOUND` in code.
These documents are finished so implementation can start immediately on the
critical path. Chetan Patel requires **short, record-winning time** — see
[../DELIVERY_SPEED.md](../DELIVERY_SPEED.md). Planning exists to go faster,
not to delay.

## 1. Order of work (never invert)

```text
1. Product decision     which bNN is first
2. Requirement spec     BUSINESS_PLATFORM/REQUIREMENTS/Bnn_*.md
3. Kernel spine map     which pNN ports the document must call
4. Metadata contract    entities, fields, layouts, validate
5. Number + process     series objects, approval definitions
6. Events + audit       facts and trail
7. Implementation       private code — not this repository
8. Evidence             tests, RTM, SoR-Live — never Production from pytest
```

If a developer opens a Python file before the requirement spec is accepted,
they are working in the wrong order.

## 2. One module at a time

Do not start `b01` through `b17` in parallel. The kernel will be forked by
seventeen half-ledgers. **Parallel unfinished modules are the slowest path.**
Record time is one posting ledger, then the next.

**Recommended first module for an enterprise posting product:** `b01_finance`.
Books are the economic law. Inventory and sales must post into them.

Alternative if the first customer cannot live without stock: `b07_inventory`,
but then a finance posting **interface** must still be specified in the same
wave (even if journals are stubbed to a port).

## 3. The spine every document must use

A JeslotERP business document is illegal if it invents its own version of:

| Need | Package | Requirement |
|---|---|---|
| Who | `p01_identity` | Permission codes `bNN.*` |
| Which company / period | `p02_organization` | Posting date vs fiscal period |
| Policy values | `p03_configuration` | Tolerances, lock flags |
| Party | `p04_business_partner` | Customer / vendor roles |
| Shape + UI | `p05_metadata` | Entity + layouts + validate |
| Labels | `p06_localization` | `label_key` only |
| Number | `p07_number_series` | Allocate once; void on discard |
| Bytes | `p08_file_media` | Attachments |
| DIR | `p09_document` | Optional controlled copy |
| Approval | `p10_process` | Inbox, not a boolean |
| Decision | `p11_rules` | Thresholds, determination |
| Flag | `p12_feature` | Risky rollout |
| Fact | `p13_event_bus` | Posted / cancelled / reversed |
| Job | `p14_messaging` | Recurring, import, depreciation |
| Notice | `p15_notification` | Task / dunning |
| Schedule | `p17_scheduler` | Period jobs |
| Trail | `p19_audit` | Field diffs on financial docs |
| Print | `p32_output` | Determination + render |
| Row ACL | `p33_sharing` | Team visibility |
| Entitlement | `p26_licensing` | Module SKU |

## 4. Requirement document — required sections

Every file in `REQUIREMENTS/` must contain:

1. Purpose and economic question
2. In scope / out of scope
3. Actors and permissions
4. Masters
5. Transactional documents (fields, statuses, numbering)
6. Posting and integrity rules
7. Processes and rules
8. Events
9. Reports
10. Platform dependency map
11. Phased delivery (MVP → statutory → advanced)
12. Acceptance criteria (testable)
13. Explicit non-claims (what we are **not** copying from any third-party ERP)

No vague line such as “implement inventory.”

## 5. Integrity laws (finance and stock)

- A journal is rejected if it does not balance.
- A posted financial document is immutable; correction is a reversing document.
- On-hand stock equals the sum of movement lines.
- Numbers are allocated once; discarded drafts void the number when policy requires.
- Closed periods reject posts; override is permissioned and audited.
- Cross-schema foreign keys are forbidden.

## 6. Definition of “finished documentation”

A business module’s documentation is finished when:

- [ ] `REQUIREMENTS/Bnn_*.md` is complete against §4
- [ ] Catalog row in `MODULE_CATALOG.md` matches the spec
- [ ] TODOs in `BUSINESS_PLATFORM_TODO.md` trace to requirement IDs
- [ ] First metadata entities are named (not yet necessarily seeded)
- [ ] First process keys and series objects are named
- [ ] MVP acceptance criteria can be turned into tests without invention

Implementation may start only after this checklist is true for the **chosen**
first module.

## 7. What serious developers contribute now

1. Review and tighten requirement specs.
2. Add missing acceptance criteria.
3. Map well-known finance objects to JeslotERP names without copying
   proprietary table designs or another vendor’s marks as our identifiers.
4. Refuse scope that belongs in the kernel or in another `bNN`.

See [REQUIREMENTS/README.md](REQUIREMENTS/README.md).
