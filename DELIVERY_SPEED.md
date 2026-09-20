# Delivery Speed — Record Time

**Owner:** Chetan Patel  
**Rule:** JeslotERP must be built in **short, record-winning time**. Too much
calendar time is a failure mode. This is a first-class product constraint,
equal to quality — not a slogan after the architecture.

## What “record time” means

- The kernel (p01–p33) already exists as a SoR-Live platform. **Do not rebuild it.**
- Business-module **requirements are finished**. **Do not rediscover scope.**
- Implement **one economic spine first** (`b01_finance`, then items/stock).
- Reuse metadata, numbers, process, rules, audit, output, sharing on every document.
- Pay serious developers to own a bounded context and **ship**, not to debate forever.
- Raise funds so the critical path can be staffed in parallel **without**
  seventeen half-modules.

Record time is **elapsed months to a posting product**, not lines of code
and not seventeen unfinished apps.

## What wastes time (forbidden)

| Waste | Why it loses the record |
|---|---|
| Starting all `b01`–`b17` at once | Seventeen stubs, zero books |
| Cloning another application-first ERP | Rebuilds the wrong architecture |
| New UI stack per document | The metadata runtime already exists |
| Redesigning p01–p33 | Kernel is the accelerator |
| Unpaid timepass / endless design | No posting date |
| Fake Production labels | Rework after a failed demo |
| Inventing `p34+` | Distraction |

## Critical path (do this, in order)

```text
Week 0     Decision locked: first module = b01_finance
           Paid owners named for finance + metadata spine
           ↓
Short wave Books post: COA, period lock, balanced journal, reverse
           Trial balance dataset
           ↓
Next wave  Items + stock movements (b07) using the same spine
           ↓
Next wave  One sales invoice and one AP invoice that POST
           (tax calculate + finance port — no PDF-only)
```

Exact calendar dates are a staffing/funding decision. The **sequence** is not
negotiable. Skipping journals to “look like a full ERP” is slower, because
every later module must be rewritten.

## How speed and quality stay together

- Integrity laws stay: balanced journal, immutable posted docs, stock = sum(movements).
- Tests on the MVP acceptance criteria in each requirement spec — no theatre.
- SoR-Live for the first ledger is the speed trophy. Production label still
  waits soak and threat review.
- Open source waits until the product is good. Dumping unfinished source
  would waste the record, not win it.

## Staffing for speed

1. Owner (Chetan Patel) — product decision, funding, accept/reject scope.
2. Paid platform engineer — keep the spine; no kernel rewrite.
3. Paid finance implementer — `b01` MVP only.
4. Next paid owner only after the previous wave posts.

If a contributor cannot work on this critical path, they are not needed yet.

The calendar shape is in [NINETY_DAY_PLAN.md](NINETY_DAY_PLAN.md).
The finish lines are in [DEFINITION_OF_DONE.md](DEFINITION_OF_DONE.md).

See [BUSINESS_PLATFORM/HOW_TO_PLAN.md](BUSINESS_PLATFORM/HOW_TO_PLAN.md) and
[BUSINESS_PLATFORM/REQUIREMENTS/B01_FINANCE.md](BUSINESS_PLATFORM/REQUIREMENTS/B01_FINANCE.md).
