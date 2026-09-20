# FAQ

## Is JeslotERP open source today?

**No.** This repository is the **public architecture**. The implementation is
private. A future source release happens only if Chetan Patel judges the
product good enough. Documentation is Apache-2.0. Source will carry its own
license if and when it is published.

## Are you competing with ERPNext and Odoo?

We acknowledge those **independent** products by name so the market is
clear. We are not cloning them, not forking them, and not affiliated with
them. They are application-first. JeslotERP is **platform-first**, aiming at
**enterprise posting discipline** with a Python kernel. See
[MARKET_POSITION.md](MARKET_POSITION.md) and [TRADEMARKS.md](TRADEMARKS.md).

## May this repository mention SAP, Microsoft Dynamics, ERPNext, and Odoo?

**As comparison only — not as our brand, and not as a legal clearance.**
The intended use is nominative: naming a real product to say we are not
that product, or to place a thesis. Forbidden: logos, “partner / certified
/ replacement,” and adjective branding such as “SAP-class.”

This FAQ is not a lawyer’s opinion. Full policy: [TRADEMARKS.md](TRADEMARKS.md).

## Can I start implementing all 17 business modules?

**No.** That is the slowest path. Record time is `b01_finance` first. See
[DELIVERY_SPEED.md](DELIVERY_SPEED.md) and [NINETY_DAY_PLAN.md](NINETY_DAY_PLAN.md).

## Is the finance module done?

**No.** The **requirement specification** is finished
([B01](BUSINESS_PLATFORM/REQUIREMENTS/B01_FINANCE.md)). Code is `PLANNED`.

## Why pay developers if the docs are public?

Because posting ledgers is skilled work and Chetan Patel wants it in
**short time**. Public docs let serious people self-select. Implementation
on the private kernel is **paid, by agreement**. Timepass is rejected.
See [FUNDING_AND_TALENT.md](FUNDING_AND_TALENT.md).

## Have you already raised funds?

This tree does **not** claim a closed round. It states the **intent** to
raise market funds so the critical path can be staffed. Ask the owner for
anything beyond that.

## Will you add p34?

Not in this phase. UoM and FX stay in `p02_organization`. New concerns need
a registry amendment, not a casual number.

## I found KaabarERP in an old copy. Is that the name?

**No.** The product name is **JeslotERP**. Old names must not be reintroduced.

## What is the single most important next artifact?

A posted, balanced journal that appears on a trial balance — on the existing
kernel, with period lock. Everything else is later.
