# Contribution Guide

**Project:** JeslotERP  
**Copyright:** 2026 Chetan Patel  
**License of this documentation:** Apache License 2.0 (`LICENSE`)

This is a **serious product**, not a hobby dump and not a place to pass time.
The owner wants this product in **short, record-winning time**. If you slow
the critical path (`b01_finance` posting MVP), you are not a fit. See
[../DELIVERY_SPEED.md](../DELIVERY_SPEED.md).
Chetan Patel is building JeslotERP to **enterprise posting depth**, raising
**market funding** to pay for that work, and will **pay qualified developers**
when contribution is at product level — not for drive-by patches.

The implementation is **private today**. Source will be considered for
**open source in the future if and only if the product is good enough**
(honest kernel, posting business modules, security review). Until then,
contributions are to this public specification and, by separate agreement,
paid work on the private kernel.

## Who we want

People who can own a bounded context and stay for the work:

- Python platform / modular monolith engineers
- ERP functional-technical architects (finance, materials, sales, or
  equivalent enterprise modules)
- Metadata, workflow, and PostgreSQL RLS practitioners
- Security and audit engineers
- Technical writers who keep IMPLEMENTED vs PLANNED honest

If you are here to rename files, copy another product’s document tree, or
“just add a table,” do not apply. That wastes everyone’s time.

## Paid contribution (intent)

| Kind of work | How it is treated |
|---|---|
| Public specification review | Welcome now; credit in the docs tree |
| Accepted requirement / architecture change | Welcome now |
| Kernel or business-module implementation | **Paid**, by agreement, on the private codebase |
| Casual / timepass PRs | Rejected |

Rates, contracts, and scopes are **not** published in this file. Serious
developers who have read the architecture and a requirement spec should
contact the project owner (Chetan Patel) with: what they have shipped, which
`pNN` or `bNN` they can own, and a short evidence list. Funding from the
market is being pursued so those engagements can scale.

No one is promised equity, a merge, or open-source commit bits from a
comment on this repository.

## How to join (now)

1. Read [FOUNDER_AIM.md](../FOUNDER_AIM.md), [FUNDING_AND_TALENT.md](../FUNDING_AND_TALENT.md),
   [MARKET_POSITION.md](../MARKET_POSITION.md), and [PLATFORM_PRINCIPLES.md](../PLATFORM_PRINCIPLES.md).
2. Read the package summary, then [`platforms/`](../platforms/README.md).
3. Read [BUSINESS_PLATFORM/HOW_TO_PLAN.md](../BUSINESS_PLATFORM/HOW_TO_PLAN.md) and one requirement spec.
4. Propose a **precise** documentation change, or request paid implementation
   work with evidence — not a generic “I can do ERP.”
5. Use canonical ids: `p05_metadata`, `b01_finance`. Never `KaabarERP`.
   Never rename packages.

## Welcome now

- Specification reviews and requirement gaps
- Traceability from TODOs to requirement IDs
- Public-safe diagrams
- Localization of documentation

## Not welcome

- Timepass, résumé spam, or unpaid “I’ll build the whole suite” claims
- Requests for private source without an agreement
- Credentials, hosts, customer stories
- Renumbering packages or inventing `p34+`
- Claiming a ledger is implemented
- Copying another vendor’s internals as the design
- Using third-party marks as JeslotERP branding (see [../TRADEMARKS.md](../TRADEMARKS.md))
- Reintroducing old product names (`KaabarERP` or private brands)

## Later (if source is opened)

Open source is a **quality gate**, not a marketing date:

- Kernel SoR-Live remains honest; at least one business ledger posts
- Threat review signed
- License for source chosen separately from this Apache documentation license
- DCO or CLA
- Paid maintainers still review kernel changes

See [../ROADMAP/OPEN_SOURCE_ROADMAP.md](../ROADMAP/OPEN_SOURCE_ROADMAP.md).

## Voice

Architect voice. Precise package ids. Explicit status. No hype. No fake
Production labels.
