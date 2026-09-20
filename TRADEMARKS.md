# Trademarks and third-party names

**Product:** JeslotERP  
**Owner:** Chetan Patel  
**Rule:** This repository is **independent**. It is not a product of SAP, Microsoft,
Frappe / ERPNext, or Odoo.

## Short answer

Using the names **SAP**, **Microsoft Dynamics**, **ERPNext**, and **Odoo** in
these documents is **intended as nominative comparison only**: to say “we
are not those products” and “we aim at the same market category.”

This page is a **publication policy**. It is **not legal advice**, not an
opinion letter, and not a warranty of non-infringement. No counsel has been
retained through this repository. If a mark owner objects, stop using that
mark and take independent advice in the relevant country.

It is **not** a claim of:

- affiliation, partnership, sponsorship, or endorsement
- compatibility, certification, or “works like” warranty
- a fork, clone, replacement, or official edition
- permission to use those companies’ logos, icons, or brand colours

If a sentence can be read as “JeslotERP *is* SAP” or “JeslotERP *is* ERPNext,”
rewrite it. JeslotERP is only JeslotERP.

## Marks that appear in this tree

The following names, product families, and related terms belong to their
owners. Mentions here do **not** transfer any right to those marks.

| Name (examples) | Typical owner (public knowledge) |
|---|---|
| SAP, S/4HANA, ECC, Fiori, ABAP, and related SAP product names | SAP SE and affiliates |
| Microsoft, Dynamics, Dynamics 365, Dataverse, Azure, and related Microsoft product names | Microsoft Corporation and affiliates |
| ERPNext, Frappe | Frappe Technologies and affiliates |
| Odoo | Odoo S.A. and affiliates |
| Salesforce, Oracle, Tally, and other named third-party systems | Their respective owners |

This table is identification, not a complete legal register. Owners may use
® or ™ on their own sites. This repository does not reproduce their logos.

## What is allowed in JeslotERP docs (policy)

1. **Identify** an existing product when the sentence needs that product’s
   real name (“ERPNext and Odoo already exist”).
2. **Compare categories** in one market-position page: application-first ERP
   versus enterprise posting discipline.
3. **Cite industry patterns** in platform guides (“number-range objects as
   seen in well-known enterprise ERPs”) so an implementer knows the *class*
   of problem. That is architecture study, not a connector or certification.
4. **Disclaim** clones, forks, and replacements.

## What is forbidden (policy)

1. **Adjective branding** — do not write “SAP-class,” “Dynamics-class,”
   “ERPNext-style,” or “Odoo-like” as if those marks were JeslotERP’s
   product grade. Say **enterprise-class** or **independent JeslotERP**.
2. **Affiliation** — no “powered by,” “SAP partner,” “official,” “certified
   for Dynamics,” or partner-badge language unless a real written agreement
   exists (none is claimed here).
3. **Replacement / compatibility claims** — do not say JeslotERP replaces
   SAP, Microsoft Dynamics, ERPNext, or Odoo, or that it imports their
   data models, DocTypes, or modules.
4. **Logos and trade dress** — do not add those companies’ logos, blue/orange
   product marks, or screenshot chrome from their UIs.
5. **Code identifiers that look like their products** — prefer JeslotERP
   names (`jeslot:`, `EXTERNAL_ERP`, `CONTENT_ARCHIVE`). Do not introduce
   new public enums such as a marketing “SAP mode.”
6. **Copying internals** — do not paste another vendor’s table names, help
   text, or copyrighted documentation.

## How to read “class of products” in `platforms/`

Some GUIDE files list well-known enterprise products as **reference
patterns** (workflow, number ranges, metadata dictionaries). Those lists
mean: “this problem is a known ERP control-plane problem.”

They do **not** mean JeslotERP implements those vendors’ products, APIs,
licenses, or certifications.

## Comparative statements

Market-position wording is Chetan Patel’s **product thesis**, not a lab
benchmark and not a statement that any named vendor is defective.

Where a limit of another product is mentioned, read it as “why JeslotERP
is being built differently,” not as a warranty about that vendor’s current
release.

## GitHub / publication

- Repository name and product name: **JeslotERP** only.
- Do not name the repo after a third-party mark.
- Do not use those marks in GitHub topics as if this were their project.

## Contact

Trademark complaints or correction requests: the copyright owner named in
[LICENSE](LICENSE) (Chetan Patel).

## Conservative review notes (not a clearance)

A cautious reading of this tree, after the 2026-09-20 pass:

| Topic | Finding |
|---|---|
| Product name / repo name | **JeslotERP** only. No third-party mark in the title. |
| Logos / trade dress | None in this tree. |
| Affiliation / partner / certified | Disclaimed on README, LICENSE.md, MARKET_POSITION. None claimed. |
| “SAP-class” / “ERPNext-style” as our grade | Removed from hero and business docs. Forbidden going forward. |
| Market table | Opinion / thesis, labelled as perception. Not a benchmark. |
| Platform GUIDE lists | Named products are **orientation**. They must not be read as “we implement that product.” |
| Vendor table codes (SNRO, T006, BAdI, …) | Must not be used as JeslotERP class names. Prefer JeslotERP words. |
| Adapter values such as an external-system code `SAP` | Nominative identifier of a third-party system, not a product mode. Do not add a marketing “SAP mode.” |

**Clearance is not given.** Marks are territorial. SAP SE, Microsoft, Frappe,
and Odoo S.A. can still send a letter. The defence, if one is needed, is
nominative / honest comparison plus the disclaimers above — not this file.

See also [LICENSE.md](LICENSE.md) and [MARKET_POSITION.md](MARKET_POSITION.md).
