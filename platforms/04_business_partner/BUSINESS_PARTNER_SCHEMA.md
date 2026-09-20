# JeslotERP Business Partner Platform — Production Schema (Advanced)

**Version:** 2.0  
**Last reviewed:** 2026-09-09  
**Status:** **Live** — full enterprise schema (58 tables) implemented in `platforms.p04_business_partner`  
**Package:** `platforms.p04_business_partner`  
**PostgreSQL schema:** `bp`  
**Companion:** [`BUSINESS_PARTNER_GUIDE.md`](BUSINESS_PARTNER_GUIDE.md) · [`BUSINESS_PARTNER_API.md`](BUSINESS_PARTNER_API.md)

> Runtime models: `platforms/p04_business_partner/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `bp` |
| Tables | `bp_*` |
| PK | UUID |
| Soft delete | partial unique on live rows |
| Cross-schema | UUID refs only |
| RLS | FORCE on tenant tables |
| OCC | `version` on `bp_partner` (+ critical children) |
| PII | mask columns / reveal audit |
| Status | `ACTIVE` \| `INACTIVE` \| `SUSPENDED` \| `MERGED` \| `DELETED` |

---

## 2. Complete table inventory (**58 tables**)

### 2.1 Lookups (12)

| # | Table | Purpose |
|---|---|---|
| 1 | `bp_address_type` | REGISTERED, BILLING, SHIPPING, … |
| 2 | `bp_contact_type` | PRIMARY, FINANCE, … |
| 3 | `bp_relationship_type` | PARENT, BILLED_TO, AGENT_OF, … |
| 4 | `bp_document_type` | PAN_CARD, GST_CERT, … |
| 5 | `bp_industry` | Industry tree |
| 6 | `bp_payment_term` | NET30, ADVANCE, … |
| 7 | `bp_category` | Categories |
| 8 | `bp_classification` | Risk class |
| 9 | `bp_group` | Hierarchical groups |
| 10 | `bp_segment` | Marketing/ops segments |
| 11 | `bp_territory` | Sales/ops territories |
| 12 | `bp_external_system` | TALLY, SAP, GSTN, … |

### 2.2 Core party (8)

| # | Table | Purpose |
|---|---|---|
| 13 | `bp_partner` | Root party |
| 14 | `bp_partner_role` | Multi-role |
| 15 | `bp_partner_company` | Company scope |
| 16 | `bp_partner_branch` | Branch scope |
| 17 | `bp_partner_alias` | Trade names / former names |
| 18 | `bp_partner_name` | Structured name parts (individual) |
| 19 | `bp_partner_identifier` | Normalized match keys |
| 20 | `bp_external_id` | External system keys |

### 2.3 Sites & contactables (7)

| # | Table | Purpose |
|---|---|---|
| 21 | `bp_site` | Operational site |
| 22 | `bp_address` | Postal addresses |
| 23 | `bp_contact` | People |
| 24 | `bp_contact_role` | Contact↔partner/site roles |
| 25 | `bp_communication_preference` | Channel prefs |
| 26 | `bp_consent` | Consent records |
| 27 | `bp_account_team` | Account managers / team |

### 2.4 Tax, identity, bank (5)

| # | Table | Purpose |
|---|---|---|
| 28 | `bp_tax_registration` | GSTIN/VAT… |
| 29 | `bp_tax_registration_address` | Tax principal place link |
| 30 | `bp_identity_document` | PAN/CIN/Aadhaar… |
| 31 | `bp_bank_account` | Banks |
| 32 | `bp_verification_event` | Verify attempts/results |

### 2.5 Relationships & network (3)

| # | Table | Purpose |
|---|---|---|
| 33 | `bp_relationship` | Partner graph edges |
| 34 | `bp_relationship_attribute` | Edge attributes |
| 35 | `bp_hierarchy_closure` | Optional closure table for fast ancestor queries |

### 2.6 Credit, risk, compliance (6)

| # | Table | Purpose |
|---|---|---|
| 36 | `bp_credit_profile` | Credit defaults |
| 37 | `bp_payment_profile` | Payment defaults |
| 38 | `bp_risk_score` | Risk score history |
| 39 | `bp_compliance_flag` | Blacklist/holds |
| 40 | `bp_exposure_snapshot` | Optional cached exposure |
| 41 | `bp_block_reason_code` | Controlled reason catalog |

### 2.7 KYC / onboarding / governance (7)

| # | Table | Purpose |
|---|---|---|
| 42 | `bp_onboarding_case` | Onboarding case |
| 43 | `bp_onboarding_checklist_item` | Checklist |
| 44 | `bp_change_request` | Pending sensitive changes |
| 45 | `bp_change_request_item` | Field-level diffs |
| 46 | `bp_approval` | Approvals |
| 47 | `bp_note` | Notes |
| 48 | `bp_attachment` | Media refs |

### 2.8 Dedupe / merge / privacy (6)

| # | Table | Purpose |
|---|---|---|
| 49 | `bp_match_candidate` | Duplicate suspects |
| 50 | `bp_match_candidate_key` | Keys that matched |
| 51 | `bp_merge_journal` | Merge audit |
| 52 | `bp_merge_journal_item` | Per-satellite actions |
| 53 | `bp_erasure_request` | GDPR-style erase |
| 54 | `bp_partner_audit` | Field-level catalog audit |

### 2.9 Plumbing (4)

| # | Table | Purpose |
|---|---|---|
| 55 | `bp_outbox` | Outbox |
| 56 | `bp_idempotency_key` | Idempotency |
| 57 | `bp_search_document` | Optional denorm search projection |
| 58 | `bp_number_reservation` | Optional code reservation before commit |

**Total: 58 tables.**

---

## 3. Key enumerations

| Enum | Values |
|---|---|
| `bp_partner_type` | ORGANIZATION, INDIVIDUAL, GOVERNMENT, INTERNAL |
| `bp_partner_role_code` | CUSTOMER, VENDOR, CONSIGNOR, CONSIGNEE, TRANSPORTER, BROKER, AGENT, WAREHOUSE_OPERATOR, DRIVER_OWNER, BOTH |
| `bp_kyc_status` | DRAFT, SUBMITTED, IN_REVIEW, APPROVED, REJECTED, ON_HOLD, EXPIRED |
| `bp_verification_status` | UNVERIFIED, PENDING, VERIFIED, FAILED, EXPIRED |
| `bp_match_status` | OPEN, CONFIRMED_DUPLICATE, DISMISSED, MERGED |
| `bp_merge_status` | PENDING, COMPLETED, FAILED, REVERSED |
| `bp_erasure_status` | REQUESTED, IN_PROGRESS, COMPLETED, REJECTED, BLOCKED_LEGAL_HOLD |
| `bp_consent_channel` | SMS, EMAIL, WHATSAPP, CALL, PUSH |
| `bp_consent_purpose` | TRANSACTIONAL, MARKETING, KYC, CREDIT_CHECK |
| `bp_change_request_status` | DRAFT, SUBMITTED, APPROVED, REJECTED, APPLIED, CANCELLED |

---

## 4. Core tables (advanced columns)

### 4.1 `bp_partner` (root) — additions beyond v1

| Column | Type | Notes |
|---|---|---|
| `lifecycle_status` | VARCHAR(30) | Align with KYC/active |
| `kyc_status` | VARCHAR(30) | |
| `data_quality_score` | NUMERIC(5,2) NULL | 0–100 |
| `match_fingerprint` | VARCHAR(128) NULL | Composite hash |
| `merged_into_partner_id` | UUID NULL | Self-ref when MERGED |
| `merged_at` | TIMESTAMPTZ NULL | |
| `is_golden` | BOOLEAN DEFAULT true | False for unresolved dupes optional |
| `segment_id` | UUID NULL | |
| `territory_id` | UUID NULL | |
| `source_channel` | VARCHAR(50) NULL | MANUAL, IMPORT, PORTAL, API |
| `onboarded_at` | TIMESTAMPTZ NULL | |
| `last_verified_at` | TIMESTAMPTZ NULL | |
| `legal_hold` | BOOLEAN DEFAULT false | |
| `anonymized_at` | TIMESTAMPTZ NULL | After erasure |
| `search_document_id` | UUID NULL | |
| + v1 columns | | code, names, pan, type, category, … |

**Indexes:** tenant+code unique active; tenant+gstin/pan via identifiers; gin/trgm optional on display_name; tenant+is_blacklisted; tenant+kyc_status.

### 4.2 `bp_partner_alias`

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `alias_type` | VARCHAR(30) | TRADE_NAME, FORMER_NAME, BRAND, LOCAL |
| `name` | VARCHAR(255) | |
| `is_primary` | BOOLEAN | |

### 4.3 `bp_partner_name` (individuals)

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID UNIQUE | |
| `salutation` | VARCHAR(20) | |
| `first_name` / `middle_name` / `last_name` | | |
| `father_name` | VARCHAR(150) NULL | Regional |

### 4.4 `bp_partner_identifier` (match keys)

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `id_type` | VARCHAR(40) | GSTIN_NORM, PAN_NORM, PHONE_E164, EMAIL_NORM, NAME_FP, ADDR_FP |
| `id_value` | VARCHAR(255) | Normalized |
| `is_verified` | BOOLEAN | |

**Unique:** active `(tenant_id, id_type, id_value)` — critical for dedupe.

### 4.5 `bp_external_id`

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `system_id` | UUID | → `bp_external_system` |
| `external_key` | VARCHAR(150) | |
| `synced_at` | TIMESTAMPTZ NULL | |
| `sync_payload` | JSONB NULL | |

**Unique:** `(tenant_id, system_id, external_key)`.

### 4.6 `bp_site`

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `site_code` | VARCHAR(50) | |
| `site_name` | VARCHAR(150) | |
| `site_type` | VARCHAR(30) | WAREHOUSE, PLANT, OFFICE, YARD, PORT |
| `primary_address_id` | UUID NULL | |
| `geo` | GEO_POINT / lat-lng | |
| `is_default_shipping` | BOOLEAN | |
| `operating_hours` | JSONB NULL | |

---

## 5. Contactables & consent

### 5.1 `bp_contact_role`

Links contact to partner and optional site with role (`BILLING`, `OPS`, `DISPATCH`).

### 5.2 `bp_consent`

| Column | Type | Notes |
|---|---|---|
| `partner_id` / `contact_id` | UUID | One required |
| `channel` | VARCHAR(20) | |
| `purpose` | VARCHAR(30) | |
| `status` | VARCHAR(20) | GRANTED/REVOKED/UNKNOWN |
| `captured_at` | TIMESTAMPTZ | |
| `evidence_media_id` | UUID NULL | |
| `valid_to` | TIMESTAMPTZ NULL | |

### 5.3 `bp_account_team`

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `user_id` | UUID | IAM |
| `team_role` | VARCHAR(40) | OWNER, CREDIT, OPS, SALES |
| `company_id` | UUID NULL | |
| `is_primary` | BOOLEAN | |

---

## 6. Tax / bank / verification

### 6.1 `bp_tax_registration` additions

`verification_status`, `verified_at`, `verified_by`, `gst_reg_type`, `e_invoice_enabled`, `e_way_bill_enabled`.

### 6.2 `bp_verification_event`

| Column | Type | Notes |
|---|---|---|
| `target_type` | VARCHAR(30) | TAX/BANK/IDENTITY/ADDRESS |
| `target_id` | UUID | |
| `provider` | VARCHAR(50) | MANUAL, GSTN, PENNY_DROP |
| `result` | VARCHAR(20) | |
| `raw_response` | JSONB NULL | Redact secrets |
| `performed_at` | TIMESTAMPTZ | |

### 6.3 `bp_bank_account` additions

`verification_status`, `penny_drop_ref`, encrypted `account_number_cipher` optional, `account_number_last4`.

---

## 7. Relationship graph

### 7.1 `bp_relationship` additions

`strength`, `is_primary_edge`, `source` (MANUAL/IMPORT), dated effectiveness.

### 7.2 `bp_hierarchy_closure`

| Column | Type | Notes |
|---|---|---|
| `ancestor_id` | UUID | |
| `descendant_id` | UUID | |
| `depth` | INT | |
| `tenant_id` | UUID | |

Maintained on hierarchy relationship changes.

---

## 8. Credit / risk / compliance

### 8.1 `bp_risk_score`

History rows: `score`, `model_key`, `factors` JSONB, `scored_at`.

### 8.2 `bp_exposure_snapshot`

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `company_id` | UUID NULL | |
| `ar_open` / `ap_open` | NUMERIC | Cached |
| `currency_code` | VARCHAR(10) | |
| `as_of` | TIMESTAMPTZ | |
| `source` | VARCHAR(30) | FINANCE_EVENT |

### 8.3 `bp_compliance_flag` additions

`block_reason_code`, `auto_released_at`, `external_case_ref`.

---

## 9. KYC / change governance

### 9.1 `bp_onboarding_case`

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `case_number` | VARCHAR(50) | |
| `status` | VARCHAR(30) | kyc status |
| `submitted_at` | TIMESTAMPTZ NULL | |
| `decided_at` | TIMESTAMPTZ NULL | |
| `decided_by` | UUID NULL | |
| `checklist_version` | VARCHAR(20) | |

### 9.2 `bp_change_request`

For sensitive edits (bank, tax, legal name) when policy requires approval.

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `status` | VARCHAR(30) | |
| `requested_by` | UUID | |
| `payload` | JSONB | Proposed state |
| `approval_id` | UUID NULL | |

---

## 10. Match / merge / erasure

### 10.1 `bp_match_candidate`

| Column | Type | Notes |
|---|---|---|
| `partner_id_a` | UUID | |
| `partner_id_b` | UUID | |
| `score` | NUMERIC(5,2) | |
| `status` | VARCHAR(30) | |
| `reasons` | JSONB | |
| `reviewed_by` | UUID NULL | |

### 10.2 `bp_merge_journal`

| Column | Type | Notes |
|---|---|---|
| `survivor_partner_id` | UUID | |
| `victim_partner_id` | UUID | |
| `status` | VARCHAR(30) | |
| `survivorship_policy_key` | VARCHAR(50) | |
| `completed_at` | TIMESTAMPTZ NULL | |
| `reversed_at` | TIMESTAMPTZ NULL | |

### 10.3 `bp_erasure_request`

| Column | Type | Notes |
|---|---|---|
| `partner_id` | UUID | |
| `status` | VARCHAR(30) | |
| `requested_by` | UUID | |
| `reason` | TEXT | |
| `blocked_reason` | TEXT NULL | legal hold |
| `completed_at` | TIMESTAMPTZ NULL | |

---

## 11. Search projection

### `bp_search_document`

Denormalized document for p18/OpenSearch:

`partner_id`, `tenant_id`, `payload` JSONB, `updated_at`.

---

## 12. RLS

All tenant tables: deny if `app.tenant_id` empty; allow bypass OR tenant match; **FORCE RLS**.  
System lookup rows (`tenant_id NULL`, `is_system`) readable with tenant context.

---

## 13. Seed minimum

- Address/contact/relationship/document types  
- External systems: MANUAL, IMPORT, GSTN  
- Block reason codes  
- Payment terms sample  
- Role codes (app enum + optional ref)  
- All `bp.*` permissions + admin grants  

---

## 14. ER overview (advanced)

```text
bp_partner
  ├── roles / company / branch / aliases / names / identifiers / external_ids
  ├── sites ── addresses
  ├── contacts ── contact_roles / consent / preferences
  ├── account_team
  ├── tax / identity / bank ── verification_events
  ├── relationships / hierarchy_closure
  ├── credit / payment / risk / exposure / compliance
  ├── onboarding / change_requests / approvals
  ├── notes / attachments
  ├── match_candidates / merge_journal
  └── erasure_requests / partner_audit
```

---

## 15. Implementation notes

1. Maintain `bp_partner_identifier` in same TX as partner/tax/contact mutations.  
2. Merge must be idempotent and emit single outbox event with survivor/victim.  
3. Never hard-delete victim if any external FK may exist — status `MERGED`.  
4. Encrypt bank account at rest when `ENCRYPTION_KEY` present.  
5. Split models: `lookups`, `partner`, `sites`, `financial`, `compliance`, `match`, `governance`, `plumbing`.  
6. Coordinate with p05 entity keys: `bp.partner`, `bp.site`, `bp.contact`, …
