# Market Position

**Product:** JeslotERP  
**Author:** Chetan Patel  
**Goal:** A next-level, **independent**, platform-first ERP at **enterprise
posting discipline** — not a faster clone of any existing application-first
ERP.

JeslotERP is **not affiliated with, endorsed by, or sponsored by** SAP,
Microsoft, Frappe / ERPNext, or Odoo. Named products below are identified
only so a reader can place this thesis in a known market. See
[TRADEMARKS.md](TRADEMARKS.md).

## The market as it is

Open, implementable ERP and licensed enterprise ERP already exist. The names
below are **nominative** (the real products people already know):

| Product (owner’s mark) | Typical strength (market perception) | Why JeslotERP is built differently |
|---|---|---|
| ERPNext | Fast functional coverage, document-centric model, community apps | Platform concerns (IAM, ALM, rules, audit, multi-company RLS, output determination) are often carried in the same application layer |
| Odoo | Broad apps, strong UX, large ecosystem | Much logic lives per-app; JeslotERP keeps ledgers off the kernel |
| SAP S/4HANA / ECC | Statutory depth, org model, finance/controlling, material ledger, transport | Typically proprietary licensing and longer change cycles |
| Microsoft Dynamics 365 | Metadata-oriented model, finance / customer-engagement split, ISV model | Typically cloud and licensing complexity |

ERPNext and Odoo show that a market exists for implementable ERP. They do
**not** prove that another document-tree clone is the next level.

JeslotERP does **not** claim to replace, certify against, or import those
products.

## JeslotERP thesis

Build the **operating system for ERP** first (33 platform packages), then
build business modules on that OS.

```text
Enterprise posting discipline
  = identity + org + metadata + numbers + process + rules
    + audit + sharing + ALM + output + integration
    + finance / inventory / tax / logistics documents

Application-first ERP (typical)
  = documents + workflows in the same application layer
```

JeslotERP refuses to put the general ledger inside `p05_metadata` or the
stock ledger inside a website controller. Ledgers are `business/bNN_*`.
Kernel packages stay reusable.

## What “next level” means here (requirements, not slogans)

1. **Metadata-driven UI and validation** — new masters/documents are seed +
   permission, not a new screen stack (`p05_metadata`).
2. **Governed numbering** — gapless and buffered legal modes (`p07_number_series`).
3. **Real process and rules** — inbox, SLA, decision tables; not a boolean
   `approved` column (`p10_process`, `p11_rules`).
4. **Immutable audit and privacy** — trail, holds, DSR (`p19_audit`, `p31_privacy`).
5. **Record ACL plus tenant RLS** — account teams without role explosion
   (`p33_sharing`).
6. **Transport / ALM** — DEV → QA → PROD as sealed packages (`p30_alm`).
7. **Output determination** — which form, language, and channel (`p32_output`).
8. **Fail-closed providers** — no fake live mail, KMS, or payments in tests.
9. **Business modules consume the kernel** — never the reverse.

## Who this project is for

Serious developers and architects who have shipped (or want to ship):

- Multi-tenant PostgreSQL + RLS systems
- Hexagonal / modular monoliths in Python
- Metadata, workflow, or financial posting engines
- Large enterprise or open ERP implementations and felt the platform gap

This is **not** a weekend theme, a CRUD starter, or a copy of another
product’s document tree under a new name.

## Company intent (Chetan Patel)

JeslotERP is a **funded-ambition product**, not a weekend demo:

1. **Build** an enterprise-class platform, then business modules that post.
2. **Raise market funds** so the work can be staffed at professional depth.
3. **Pay serious developers** for product-level contribution (see the
   contribution guide). Casual / timepass work is not accepted.
4. **Open-source the implementation later only if the product is very good**
   — honest kernel, real ledgers, security review. Documentation is public
   now so the architecture can be judged. Source is not a trophy dump.
5. **Win on elapsed time.** Do not spend years rediscovering the product.
   Specs and kernel are done so implementation can be **short, record-winning
   time** — one spine, paid owners, no timepass. See [DELIVERY_SPEED.md](DELIVERY_SPEED.md).

The public name of this product is **JeslotERP**. Do not use other names.

## Honest present tense

- Kernel p01–p33: SoR-Live documentation and private implementation exist.
- Business modules b01–b17: **specified here, not implemented**.
- Source code: **private** until quality and a formal release decision.
- This repository: public architecture for serious review, planning, and
  paid engagement — not for timepass.

See [BUSINESS_PLATFORM/HOW_TO_PLAN.md](BUSINESS_PLATFORM/HOW_TO_PLAN.md) and
[DEVELOPMENT/CONTRIBUTION_GUIDE.md](DEVELOPMENT/CONTRIBUTION_GUIDE.md).
