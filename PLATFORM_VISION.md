# Platform Vision

**Author:** Chetan Patel  
**Ambition:** Independent **enterprise-class** ERP, built as a Python
platform, then as business modules. Existing application-first ERPs are
acknowledged competitors, not templates. JeslotERP is **unaffiliated** with
SAP, Microsoft, ERPNext, or Odoo — see [TRADEMARKS.md](TRADEMARKS.md).

## Horizon

JeslotERP should become the public architecture — and later a community
platform — on which serious teams build horizontal ERP and industry packs
**without forking** identity, workflow, metadata, audit, or numbering.

The implementation is private today. Chetan Patel is serious about this
product: market funding, paid contributors at professional level, and
**open source later only if quality justifies it**. This repository exists
so architects can review the kernel, finish business-module requirements,
and apply as serious contributors — not as timepass.

## Why not another application-first ERP

Those products already cover many documents. The next-level gap is a real
**platform OS**: layered metadata, process/rules, ALM, sharing, output
determination, and fail-closed multi-tenancy. JeslotERP spends its uniqueness
there. Business documents come second, on that OS.

See [MARKET_POSITION.md](MARKET_POSITION.md) and [FOUNDER_AIM.md](FOUNDER_AIM.md).

## Near term (documentation is done — implement the spine at record speed)

Requirements for b01–b17 are finished. Kernel specs are published. The
remaining work is **paid implementation on the critical path** in the shortest
honest calendar. See [DELIVERY_SPEED.md](DELIVERY_SPEED.md).

1. Finish and keep honest the 33 platform specifications (`PACKAGES/` + `platforms/`).
2. Finish **all** business-module requirement specifications (this wave).
3. Choose **one** first ledger: recommended `b01_finance` (books before
   commerce) or `b07_inventory` if stock truth is the commercial gate.
4. Implement that module on the private kernel using the requirement spec —
   not a parallel prototype.

## Medium term (economic core)

- Finance close + inventory valuation + tax calculate.
- Sales and purchase that **post**, not only print.
- Treasury payments against open items.

## Long term (suite + ecosystem)

- Manufacturing, warehouse, logistics, assets, projects, CRM, service, HCM.
- Sealed industry packs via `p30_alm`.
- Partner connectors via `p23_integration`.
- Assistive AI that never bypasses process, rules, or audit.

## Non-goals

- Cloning another product’s document tree or models into `pNN_*`.
- Re-implementing a full licensed enterprise suite **inside the kernel**.
- Inventing platform numbers beyond p33 in this phase.
- Claiming the product is already open source, or opening source before it is good.
- Treating this repository as unpaid entertainment.
- Shipping fake live vendors in automated tests.
- Starting all seventeen business modules at once.
