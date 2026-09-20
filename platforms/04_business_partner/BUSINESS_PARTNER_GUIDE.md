# JeslotERP Business Partner Platform — Developer Integration Guide

**Version:** 2.0 (Advanced)  
**Last reviewed:** 2026-09-09  
**Status:** **Live (complete)** — full schema (58 tables), services, and API per Guide/Schema/API v2  
**Package:** `platforms.p04_business_partner`  
**PostgreSQL schema:** `bp`  
**Depends on:** `p01_identity`, `p02_organization`, `p03_configuration`  
**Optionally integrates:** `p05_metadata`, `p06_localization`, `p08_file_media`, `p10_process`, `p15_notification`, `p18_search`, `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`BUSINESS_PARTNER_SCHEMA.md`](BUSINESS_PARTNER_SCHEMA.md) · [`BUSINESS_PARTNER_API.md`](BUSINESS_PARTNER_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-09 | Initial party master + roles + satellites. |
| **2.0** | **2026-09-09** | Advanced upgrade: party model depth, golden-record/dedupe/merge, KYC onboarding, sites, aliases, external IDs, credit risk, consent, governance, match engine, erase/GDPR hooks. |
| **2.1** | **2026-09-12** | P33-LIVE-002: list partners share-filters when p33 grants exist. |
| **2.2** | **2026-09-12** | P33-LIVE-003: public partner-scoped reads evaluate p33. |

---

## 1. Purpose (advanced)

`p04_business_partner` is JeslotERP’s **enterprise party / counterparty control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- SAP Business Partner + CVI  
- Oracle Trading Community Architecture (TCA)  
- Microsoft Dynamics Account / Contact / Party  
- Salesforce Account hierarchy + duplicate rules  

It is **not** a thin customer table. It is the system of record for **who we trade and ship with**, across every role, company, and compliance state.

### Owns

| Domain | Examples |
|---|---|
| Party master | Organization / individual / government parties |
| Roles | Customer, vendor, consignor, consignee, transporter, broker, agent, … |
| Commercial profile | Category, group, segment, score, account team |
| Sites & addresses | Operational sites, geo, billing/shipping |
| Contacts & consent | People, preferences, communication consent |
| Tax & identity | GSTIN/VAT, PAN/CIN, verification state |
| Banking | Accounts, verification, mask rules |
| Relationships | Hierarchy, billed-to, agent-of, network graph |
| Credit & risk | Limits, ratings, holds (defaults — not ledgers) |
| Compliance | Blacklist, watchlist, KYC cases, legal hold |
| Data quality | Dedupe candidates, golden record, merge |
| Onboarding | Journeys, checklists, approvals |
| Integration | External system IDs, sync stamps |
| Governance | Change requests, approvals, audit |

### Does **not** own

| Concern | Owner |
|---|---|
| Users / passwords / sessions | `p01_identity` (optional `user_id` link only) |
| Tenant / company / branch masters | `p02_organization` |
| Setting values | `p03_configuration` |
| Field/form catalogs | `p05_metadata` (BP entity described there) |
| Translations | `p06_localization` |
| Invoices / AR-AP balances / payments | Finance modules |
| Bilty / LR / challan documents | Transport modules |
| File bytes | `p08_file_media` |

**One legal party → many roles.** Never duplicate “Customer Acme” and “Vendor Acme” as two masters.

---

## 2. Architectural position

```text
p01 ──► p02 ──► p03 ──► p04 business_partner ◄── p05 metadata (descriptors)
                           │
     ┌──────────┬──────────┼──────────┬──────────┬──────────┐
     ▼          ▼          ▼          ▼          ▼          ▼
   Sales     Purchase   Logistics   Finance    Search      Portal
  (AR party) (AP party) (ship roles) (limits)  (match)   (BP users)
```

**Hard rules**

1. Business modules store `partner_id` (+ optional role snapshot).  
2. Never invent parallel customer/vendor master tables.  
3. Blacklisted / blocked partners fail closed on **new** documents.  
4. Cross-platform access via gateway / `/internal/v1/bp` — never BP ORM imports.  
5. No cross-schema FKs.

---

## 3. Advanced design principles

1. **Party-centric aggregate** — `bp_partner` root; roles/sites/tax are satellites.  
2. **Role cardinality** — many active roles; role-effective dating; company-scoped roles.  
3. **Site model** — operational sites distinct from postal addresses when needed.  
4. **Golden record** — duplicate detection + survivorship + merge with audit.  
5. **Temporal validity** — roles, tax, relationships, credit can be dated.  
6. **External identity map** — stable keys for GST portal, Tally, SAP, etc.  
7. **KYC / onboarding state machine** — not just a boolean.  
8. **Consent & preference** — channel opt-in/out for notifications.  
9. **PII minimization** — mask PAN/bank/Aadhaar; reveal permissions.  
10. **Change governance** — sensitive updates can require approval.  
11. **Match & search** — normalized keys + fuzzy candidates for dedupe.  
12. **CQRS + outbox** — thin HTTP; rich application services.  
13. **RLS fail-closed + FORCE**.  
14. **Idempotent create + OCC** on partner root.  
15. **Metadata-aware** — `custom_fields` validated via p05 when Live.  
16. **Erase / legal hold** — GDPR-style request workflow (tombstone, not silent hard delete of audit).

---

## 4. Party model

### 4.1 Partner types

| Type | Meaning |
|---|---|
| `ORGANIZATION` | Company / LLP / firm |
| `INDIVIDUAL` | Person |
| `GOVERNMENT` | Govt / PSU |
| `INTERNAL` | Rare internal trading party (prefer ORG company) |

### 4.2 Roles (canonical)

| Code | Use |
|---|---|
| `CUSTOMER` | Pays / AR |
| `VENDOR` | Supplier / AP |
| `CONSIGNOR` | Shipper |
| `CONSIGNEE` | Receiver |
| `TRANSPORTER` | Carrier / hire |
| `BROKER` | Commission |
| `AGENT` | Booking agent |
| `WAREHOUSE_OPERATOR` | 3PL |
| `DRIVER_OWNER` | Optional fleet owner party |
| `BOTH` | Legacy only — migrate to multi-role |

### 4.3 Effective partner for a transaction

Resolver inputs: `tenant_id`, `company_id`, `required_role`, `as_of_date`.  
Checks: exists, not deleted, active, role effective, company scoped, not blocked (`BLACKLIST`/`PAYMENT_HOLD`/`LEGAL_HOLD` with `BLOCK` severity).

---

## 5. Golden record, dedupe & merge

### 5.1 Match keys (normalized)

Examples: normalized GSTIN, PAN, phone E.164, email, legal name fingerprint, PIN+line1 hash.

### 5.2 Candidate generation

`bp_match_candidate` stores suspected duplicates with score + reason codes (`SAME_GSTIN`, `SAME_PAN`, `FUZZY_NAME`, …).

### 5.3 Merge

- Survivor partner keeps `id`  
- Victim soft-merged (`status=MERGED`, `merged_into_partner_id`)  
- Satellites re-linked or closed per survivorship rules  
- `bp_merge_journal` immutable audit  
- Outbox `bp.partner.merged` so consumers rewire FKs  

### 5.4 Survivorship policy

Configurable per field group: prefer verified tax > newest > non-null > survivor.

---

## 6. KYC & onboarding

States on partner / case:

`DRAFT → SUBMITTED → IN_REVIEW → APPROVED → ACTIVE`  
or `REJECTED` / `ON_HOLD`

`bp_onboarding_case` + checklist items + document requirements.  
Can integrate `p10_process` for approvals when Live; until then BP-native approval table.

---

## 7. Credit & risk (master only)

BP stores **policy defaults**, not open items:

- Credit limit, days, currency  
- Rating / risk score  
- Block-on-limit / allow-overdue flags  
- Exposure snapshot **optional cache** (updated by Finance events) — never journal truth  

Finance remains authoritative for balances.

---

## 8. Compliance & privacy

| Capability | Behavior |
|---|---|
| Blacklist / watchlist | Flags + partner denorm `is_blacklisted` |
| Legal hold | Blocks erase/merge |
| Erasure request | `bp_erasure_request` workflow; anonymize PII; keep id for FK integrity |
| Consent | Per channel marketing/transactional |
| Masking | Bank/PAN/Aadhaar masked unless reveal permission |

---

## 9. Multi-tenancy & scope

| Layer | Rule |
|---|---|
| Tenant | Partner owned by one tenant |
| Company | `bp_partner_company` visibility |
| Branch | Optional `bp_partner_branch` |
| Site | Operational locations under partner |
| Territory | Optional territory assignment for sales/ops |

JWT / context-switch only — never `X-Tenant-ID` authority.

---

## 10. Security contract

### Permissions (advanced)

| Code | Use |
|---|---|
| `bp.partner.read` | List/get |
| `bp.partner.create` | Create |
| `bp.partner.update` | Update |
| `bp.partner.delete` | Soft-delete |
| `bp.partner.merge` | Merge duplicates |
| `bp.partner.blacklist` | Compliance block flags |
| `bp.partner.approve` | Onboarding / change approval |
| `bp.partner.erase` | Erasure workflow |
| `bp.role.assign` | Roles |
| `bp.address.manage` / `bp.site.manage` | Address/site |
| `bp.contact.manage` | Contacts |
| `bp.bank.manage` | Banks (+ reveal) |
| `bp.tax.manage` | Tax |
| `bp.relationship.manage` | Graph |
| `bp.credit.manage` | Credit/risk |
| `bp.kyc.manage` | KYC cases |
| `bp.match.manage` | Dedupe review |
| `bp.category.manage` / `classification` / `group` | Lookups |
| `bp.settings.manage` | System lookups |
| `bp.export` | Bulk export |
| `bp.*` | Wildcard |

### Trust

- IAM enforces permissions.  
- Internal token for mesh validate/resolve.  
- Reveal endpoints audited.

---

## 11. Domain events (outbox)

| Event | When |
|---|---|
| `bp.partner.created` / `updated` / `activated` / `deactivated` / `deleted` | Lifecycle |
| `bp.partner.role_assigned` / `role_revoked` | Roles |
| `bp.partner.blacklisted` / `unblacklisted` | Compliance |
| `bp.partner.merged` | Golden record |
| `bp.partner.kyc_status_changed` | KYC |
| `bp.partner.credit_changed` | Credit profile |
| `bp.partner.match_candidate_opened` / `resolved` | Dedupe |
| `bp.tax.verified` / `bp.bank.verified` | Verification |
| `bp.partner.erasure_requested` / `erasure_completed` | Privacy |
| `bp.site.changed` / `bp.address.changed` | Locations |

Stream: `jesloterp:business_partner:outbox`.

---

## 12. Module layout (advanced)

```text
platforms/p04_business_partner/
  application/
    commands/…  queries/…
    services/
      effective_partner.py      # role+scope+block checks
      match_engine.py           # fingerprint + candidates
      merge_service.py
      kyc_workflow.py
      credit_policy.py
      redaction.py
      erasure_service.py
    permissions/catalog.py
    errors.py                   # BpAppError
  domain/
    aggregates/partner.py
    policies/survivorship.py
    events/…
  infrastructure/
    http/… persistence/… messaging/outbox/
    adapters/normalization.py   # GSTIN/PAN/phone normalize
  tests/
```

Load: Identity → Organization → Configuration → **BusinessPartner** → Metadata…

---

## 13. Integration rules

1. Transport/Finance call `POST /internal/v1/bp/partners/validate-for-use`.  
2. Snapshot display fields on documents; keep `partner_id`.  
3. Subscribe to `bp.partner.merged` to rewrite references.  
4. Validate `custom_fields` with p05 when available.  
5. Notifications honor consent preferences via p15.  
6. Search indexes match keys via p18.

---

## 14. Build phases (advanced)

| Phase | Deliverable | Status |
|---|---|---|
| P0 | Docs v2 (this set) | Done |
| P1 | Skeleton, RLS, permissions, lookups | Done |
| P2 | Partner + roles + company/branch scope | Done |
| P3 | Address/contact/bank/tax/identity | Done |
| P4 | Relationships + sites + aliases + external IDs | Done |
| P5 | Credit/compliance/consent | Done |
| P6 | KYC onboarding + approvals | Done |
| P7 | Match engine + merge | Done |
| P8 | Erasure + catalog audit + export | Done |
| P9 | Internal mesh hardening + search hooks | Done |
| P10 | Registry → **Live** | Done |

---

## 15. Definition of Done (advanced)

- [x] Multi-role + company scope enforced on validate-for-use  
- [x] Blacklist/hold fail-closed for new docs  
- [x] Bank/PAN masked by default; reveal audited  
- [x] Dedupe candidates + merge journal + outbox  
- [x] KYC state machine with approval  
- [x] External ID uniqueness per system  
- [x] Erasure respects legal hold  
- [x] OCC + idempotent create  
- [x] FORCE RLS + unit/contract tests  
- [x] P33-LIVE-002: list partners share-filters (skip if no grants; drop stranger rows)  
- [x] P33-LIVE-003: public partner-scoped reads evaluate (GET + validate-for-use)  
- [x] No cross-schema FKs; gateway-only consumers  

---

## 16. Anti-patterns

| Don’t | Do |
|---|---|
| Separate customer & vendor masters | One partner + roles |
| Trust headers for tenant | JWT / context switch |
| Store AR balances in BP | Finance events / ledgers |
| Hard-delete merged victim without journal | Merge journal + soft MERGED |
| Return full account numbers in lists | Mask + reveal permission |
| Skip validate-for-use in bilty create | Internal mesh check |
| Import BP ORM from CRM | Gateway / internal API |

---

## 17. Related documents

- Schema v2: [`BUSINESS_PARTNER_SCHEMA.md`](BUSINESS_PARTNER_SCHEMA.md)  
- API v2: [`BUSINESS_PARTNER_API.md`](BUSINESS_PARTNER_API.md)  
- Metadata: [`../05_metadata/METADATA_GUIDE.md`](../05_metadata/METADATA_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
