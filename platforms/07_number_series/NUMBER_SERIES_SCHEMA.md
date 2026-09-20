# JeslotERP Number Series Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-09  
**Status:** **Live** — 67 ORM tables + Alembic `c1d2e3f4a5b6` / RLS `d2e3f4a5b6c7` (`schema=number_series`)  
**Package:** `platforms.p07_number_series`  
**PostgreSQL schema:** `number_series`  
**Companion:** [`NUMBER_SERIES_GUIDE.md`](NUMBER_SERIES_GUIDE.md) · [`NUMBER_SERIES_API.md`](NUMBER_SERIES_API.md)

> Runtime models: `platforms/p07_number_series/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `number_series` (never `p07`) |
| Tables | `ns_*` |
| Object keys | Upper snake (`TAX_INVOICE`, `BILTY`) |
| Soft delete | Partial uniques on live rows |
| Cross-schema | UUID refs only (org company/branch/fiscal) |
| RLS | FORCE on tenant-scoped counters & ledger |
| Numbers | Store `formatted_number` + `sequence_value` + `allocation_id` |
| Immutability | Issued ledger rows never hard-deleted |

---

## 2. Complete table inventory (**64 tables**)

### 2.1 Catalog & document binding (8)

| # | Table | Purpose |
|---|---|---|
| 1 | `ns_document_type` | Logical doc kinds (order, invoice, …) |
| 2 | `ns_series_object` | Number-range objects |
| 3 | `ns_series_object_alias` | Deprecated keys → canonical |
| 4 | `ns_document_binding` | object ↔ consuming module/entity |
| 5 | `ns_series_group` | Admin grouping |
| 6 | `ns_series_group_member` | M2M |
| 7 | `ns_channel` | WEB/API/IMPORT/MOBILE issue channels |
| 8 | `ns_feature_binding` | Optional feature-flag gating |

### 2.2 Definitions & segments (10)

| # | Table | Purpose |
|---|---|---|
| 9 | `ns_series_definition` | Versioned definition header |
| 10 | `ns_segment_type` | CONSTANT, SEQUENCE, COMPANY_CODE, … |
| 11 | `ns_series_segment` | Ordered segments on a definition |
| 12 | `ns_format_pattern` | Cached/derived format + regex |
| 13 | `ns_check_digit_rule` | Luhn/Mod97/custom |
| 14 | `ns_charset_rule` | Allowed charset for external |
| 15 | `ns_validation_rule` | Extra validation predicates |
| 16 | `ns_series_template` | Reusable template |
| 17 | `ns_series_template_item` | Template segments |
| 18 | `ns_definition_activation` | Which definition is live per assignment |

### 2.3 Scope & assignment (7)

| # | Table | Purpose |
|---|---|---|
| 19 | `ns_scope_dimension` | TENANT, COMPANY, BRANCH, FISCAL_YEAR, … |
| 20 | `ns_series_scope` | Dimensions required by object/definition |
| 21 | `ns_series_assignment` | Resolves object+scope → definition/interval set |
| 22 | `ns_company_series_binding` | Company defaults |
| 23 | `ns_branch_series_binding` | Branch defaults |
| 24 | `ns_tenant_series_override` | Tenant policy overrides |
| 25 | `ns_assignment_priority` | Conflict resolution weights |

### 2.4 Intervals, counters, buffers (8)

| # | Table | Purpose |
|---|---|---|
| 26 | `ns_range_interval` | From/to/current for a scope key |
| 27 | `ns_range_period` | FY/period binding on interval |
| 28 | `ns_counter_state` | Hot counter row (lock target) |
| 29 | `ns_concurrency_profile` | Gapless vs buffered settings |
| 30 | `ns_buffer_pool` | Leased numeric ranges |
| 31 | `ns_buffer_lease` | Instance lease records |
| 32 | `ns_buffer_checkpoint` | High-water checkpoints |
| 33 | `ns_interval_extension` | Approved to-value extensions |

### 2.5 Allocation runtime (10)

| # | Table | Purpose |
|---|---|---|
| 34 | `ns_allocation` | Issued/reserved ledger |
| 35 | `ns_allocation_segment_value` | Materialized segment values |
| 36 | `ns_reservation` | Reservation header |
| 37 | `ns_reservation_item` | Reserved numbers |
| 38 | `ns_void_record` | Void reasons |
| 39 | `ns_recycle_bin` | Recyclable candidates |
| 40 | `ns_external_intake` | Manual/external registrations |
| 41 | `ns_collision_log` | Duplicate attempts |
| 42 | `ns_manual_override_request` | Ask to use manual on hybrid |
| 43 | `ns_manual_override_grant` | Approval grant |

### 2.6 Legal, thresholds, fiscal ops (8)

| # | Table | Purpose |
|---|---|---|
| 44 | `ns_legal_policy` | Gapless/no-reuse/etc. |
| 45 | `ns_legal_policy_binding` | Bind policy → object/assignment |
| 46 | `ns_threshold_rule` | % or remaining-count alerts |
| 47 | `ns_threshold_event` | Emitted breaches |
| 48 | `ns_fiscal_binding` | Object ↔ fiscal calendar ref |
| 49 | `ns_period_reset_rule` | Rollover rules |
| 50 | `ns_rollover_job` | Job definition |
| 51 | `ns_rollover_run` | Execution history |

### 2.7 Governance & packs (7)

| # | Table | Purpose |
|---|---|---|
| 52 | `ns_changeset` | Definition change batch |
| 53 | `ns_changeset_item` | Ops |
| 54 | `ns_approval` | Approvals |
| 55 | `ns_publish_version` | Activation versions |
| 56 | `ns_series_package` | Numbering packs |
| 57 | `ns_series_package_item` | Pack payload refs |
| 58 | `ns_simulate_run` | Dry-run simulations |

### 2.8 Quality / migration / plumbing (6)

| # | Table | Purpose |
|---|---|---|
| 59 | `ns_gap_scan` | Gap scan header |
| 60 | `ns_gap_finding` | Missing/unexpected numbers |
| 61 | `ns_import_batch` | Legacy import |
| 62 | `ns_import_item` | Import rows |
| 63 | `ns_reconcile_run` | Ledger vs domain reconcile |
| 64 | `ns_usage_stats` | Aggregated usage |

**Plus plumbing (counted in implementation as siblings; keep in same schema):**

| Table | Purpose |
|---|---|
| `ns_outbox` | Outbox |
| `ns_idempotency_key` | Allocate idempotency |
| `ns_series_audit` | Catalog/ledger audit |

> Inventory focus: **64 domain tables** above; plumbing is mandatory in package but listed separately so object counts stay comparable to p04–p06.

**Implementation total with plumbing: 67 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `ns_allocation_mode` | `CONTINUOUS_GAPLESS`, `NON_CONTINUOUS_BUFFERED`, `EXTERNAL_MANUAL`, `HYBRID` |
| `ns_allocation_status` | `RESERVED`, `ISSUED`, `VOIDED`, `RECYCLED`, `EXPIRED` |
| `ns_segment_kind` | `CONSTANT`, `SEPARATOR`, `SEQUENCE`, `COMPANY_CODE`, `BRANCH_CODE`, `FISCAL_YEAR`, `CALENDAR_YEAR`, `MONTH`, `DAY`, `DOC_SUBTYPE`, `CHANNEL`, `FREE_TEXT`, `CHECK_DIGIT`, `RANDOM_TOKEN` |
| `ns_scope_dim` | `SHARED`, `TENANT`, `COMPANY`, `BRANCH`, `FISCAL_YEAR`, `DOC_SUBTYPE`, `CHANNEL` |
| `ns_issue_source` | `INTERNAL`, `EXTERNAL`, `RECYCLE`, `IMPORT` |
| `ns_check_digit_algo` | `NONE`, `LUHN`, `MOD97`, `MOD11`, `CUSTOM_WEIGHT` |
| `ns_threshold_metric` | `PCT_USED`, `REMAINING`, `VOID_RATE` |
| `ns_definition_lifecycle` | `DRAFT`, `ACTIVE`, `SUPERSEDED`, `RETIRED` |
| `ns_interval_status` | `OPEN`, `EXHAUSTED`, `CLOSED`, `PENDING_ROLLOVER` |

---

## 4. Catalog detail

### 4.1 `ns_series_object`

| Column | Type | Notes |
|---|---|---|
| `object_key` | VARCHAR(50) UNIQUE | `TAX_INVOICE` |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `document_type_id` | UUID NULL | |
| `default_mode` | VARCHAR(30) | |
| `is_system` | BOOLEAN | Seeded product object |
| `is_active` | BOOLEAN | |
| `label_key` | VARCHAR(200) NULL | i18n |

### 4.2 `ns_document_binding`

| Column | Type | Notes |
|---|---|---|
| `object_id` | UUID | |
| `module_key` | VARCHAR(50) | `transport`, `finance` |
| `entity_key` | VARCHAR(100) | metadata entity key |
| `is_primary` | BOOLEAN | |

### 4.3 `ns_series_definition`

| Column | Type | Notes |
|---|---|---|
| `object_id` | UUID | |
| `version_number` | INT | |
| `lifecycle` | VARCHAR(20) | |
| `allocation_mode` | VARCHAR(30) | |
| `sequence_start` | BIGINT | Default 1 |
| `sequence_end` | BIGINT NULL | Null = unbounded soft |
| `sequence_step` | INT DEFAULT 1 | |
| `pad_length` | INT | For SEQUENCE segment default |
| `pad_char` | CHAR(1) DEFAULT '0' | |
| `prefix_legacy` | VARCHAR(50) NULL | Compat |
| `suffix_legacy` | VARCHAR(50) NULL | |
| `allow_peek` | BOOLEAN | |
| `reservation_ttl_seconds` | INT NULL | |
| `checksum` | VARCHAR(64) | Definition content hash |

**Unique:** `(object_id, version_number)`.

---

## 5. Segments

### 5.1 `ns_series_segment`

| Column | Type | Notes |
|---|---|---|
| `definition_id` | UUID | |
| `position` | INT | 1..n |
| `segment_kind` | VARCHAR(30) | |
| `literal_value` | VARCHAR(50) NULL | CONSTANT/SEPARATOR |
| `value_source` | VARCHAR(50) NULL | e.g. `scope.company_code` |
| `format_spec` | VARCHAR(50) NULL | `YY`, `YYYY`, `YYYY`, `MM` |
| `pad_length` | INT NULL | Override |
| `max_length` | INT NULL | |
| `transform` | VARCHAR(30) NULL | `UPPER`, `LOWER`, `TRIM` |
| `check_digit_rule_id` | UUID NULL | If kind=CHECK_DIGIT |
| `is_required` | BOOLEAN | |

### 5.2 Fiscal format specs

| Spec | Example |
|---|---|
| `YYYY` | `2025` |
| `YY` | `25` |
| `YYYY` | `2526` (Indian FY compact) |
| `FY_LABEL` | `2025-26` |
| `MM` / `DD` | Calendar parts from posting date |

---

## 6. Scope & assignment

### 6.1 `ns_series_assignment`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | |
| `object_id` | UUID | |
| `company_id` | UUID NULL | |
| `branch_id` | UUID NULL | |
| `fiscal_year_id` | UUID NULL | Org ref |
| `doc_subtype` | VARCHAR(50) NULL | |
| `definition_id` | UUID | Active definition |
| `legal_policy_id` | UUID NULL | |
| `concurrency_profile_id` | UUID NULL | |
| `priority` | INT | Higher wins |
| `is_active` | BOOLEAN | |
| `valid_from` / `valid_to` | TIMESTAMPTZ | |

**Unique live scope:** partial unique on non-null scope tuple + object + tenant.

### 6.2 Scope key materialization

Allocator computes `scope_hash` = sha256 of normalized scope JSON for counter lookup.

---

## 7. Intervals & counters

### 7.1 `ns_range_interval`

| Column | Type | Notes |
|---|---|---|
| `assignment_id` | UUID | |
| `scope_hash` | VARCHAR(64) | |
| `from_value` | BIGINT | |
| `to_value` | BIGINT | Inclusive |
| `current_value` | BIGINT | Last issued (or 0) |
| `status` | VARCHAR(20) | |
| `fiscal_year_id` | UUID NULL | |
| `opened_at` / `closed_at` | TIMESTAMPTZ | |

### 7.2 `ns_counter_state`

Hot row: `(interval_id, current_value, updated_at, lock_version)`.  
Gapless updates under row lock; buffered updates via buffer leases.

### 7.3 `ns_buffer_pool` / `ns_buffer_lease`

| Column | Type | Notes |
|---|---|---|
| `interval_id` | UUID | |
| `range_from` / `range_to` | BIGINT | |
| `leased_to_instance` | VARCHAR(100) | |
| `leased_at` | TIMESTAMPTZ | |
| `expires_at` | TIMESTAMPTZ | |
| `high_water` | BIGINT | |
| `status` | VARCHAR(20) | ACTIVE/EXHAUSTED/REVOKED |

---

## 8. Allocation ledger

### 8.1 `ns_allocation`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | RLS |
| `object_id` | UUID | |
| `assignment_id` | UUID | |
| `definition_id` | UUID | |
| `interval_id` | UUID | |
| `company_id` / `branch_id` / `fiscal_year_id` | UUID NULL | Denorm scope |
| `sequence_value` | BIGINT | Raw sequence |
| `formatted_number` | VARCHAR(100) | Final display |
| `status` | VARCHAR(20) | |
| `source` | VARCHAR(20) | INTERNAL/EXTERNAL/… |
| `reservation_id` | UUID NULL | |
| `document_ref_type` | VARCHAR(50) NULL | Consumer hint |
| `document_ref_id` | UUID NULL | |
| `issued_at` | TIMESTAMPTZ | |
| `issued_by` | UUID NULL | User |
| `idempotency_key` | VARCHAR(100) NULL | |
| `scope_hash` | VARCHAR(64) | |
| `check_digit` | VARCHAR(10) NULL | |

**Uniques (partial, live):**

- `(tenant_id, object_id, scope_hash, formatted_number)` where status in (RESERVED, ISSUED)  
- `(tenant_id, idempotency_key)` where key not null  

### 8.2 `ns_reservation`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `object_id` | UUID | |
| `expires_at` | TIMESTAMPTZ | |
| `document_ref_id` | UUID NULL | |
| `status` | VARCHAR(20) | OPEN/COMMITTED/RELEASED/EXPIRED |
| `created_by` | UUID | |

### 8.3 Void / recycle

`ns_void_record`: `allocation_id`, `reason_code`, `reason_text`, `voided_by`, `voided_at`.  
`ns_recycle_bin`: only if legal policy allows; recycle allocates same formatted number with new allocation row linking `recycled_from_id`.

---

## 9. Legal policy

### 9.1 `ns_legal_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | `IN_GST_TAX_INVOICE` |
| `require_gapless` | BOOLEAN | |
| `forbid_reuse` | BOOLEAN | |
| `require_fiscal_scope` | BOOLEAN | |
| `require_company_scope` | BOOLEAN | |
| `allow_manual_override` | BOOLEAN | |
| `max_void_rate_pct` | NUMERIC(5,2) NULL | |
| `retain_allocation_years` | INT NULL | |

---

## 10. Thresholds & rollover

### 10.1 `ns_threshold_rule`

| Column | Type | Notes |
|---|---|---|
| `interval_id` OR `assignment_id` | UUID | |
| `metric` | VARCHAR(20) | PCT_USED / REMAINING |
| `threshold_value` | NUMERIC | e.g. 80 or remaining 1000 |
| `notify_channel` | VARCHAR(30) | outbox/event |

### 10.2 `ns_period_reset_rule`

| Column | Type | Notes |
|---|---|---|
| `object_id` | UUID | |
| `reset_on` | VARCHAR(30) | FISCAL_YEAR_OPEN |
| `new_from_value` | BIGINT DEFAULT 1 | |
| `new_to_value` | BIGINT NULL | |
| `carry_prefix_strategy` | VARCHAR(30) | REBUILD_SEGMENTS |

---

## 11. Governance & packs

- `ns_changeset` / `ns_changeset_item` / `ns_approval` — definition edits  
- `ns_publish_version` — activation stamp + checksum  
- `ns_series_package` — `india.gst.tax_invoice@1.0.0` with items seeding object+policy+template  
- `ns_simulate_run` — inputs/outputs JSON for admin dry-run  

---

## 12. Quality & migration

### 12.1 Gap scan

`ns_gap_scan` + `ns_gap_finding`: for continuous series, find missing sequence values between min/max issued; classify EXPECTED_VOID vs ANOMALY.

### 12.2 Import

`ns_import_batch` loads legacy numbers into ledger as `source=IMPORT` without advancing wrongly; sets `current_value` to max imported when `sync_counter=true`.

### 12.3 Reconcile

`ns_reconcile_run` compares domain document numbers vs ledger via gateway samples / hash reports.

---

## 13. Plumbing

| Table | Purpose |
|---|---|
| `ns_outbox` | Domain events |
| `ns_idempotency_key` | Request dedupe payload |
| `ns_series_audit` | Before/after JSON for catalog + sensitive ops |

---

## 14. RLS summary

| Class | Policy |
|---|---|
| System objects/templates | Read auth; manage permission |
| Assignments, intervals, counters, buffers | FORCE `tenant_id` |
| Allocations, reservations, voids | FORCE `tenant_id` |
| Packages system | Readable; install permission |

---

## 15. Seed minimum

1. Segment types + scope dimensions  
2. Objects: `BILTY`, `LOADING_SLIP`, `FREIGHT_CHALLAN`, `TRIP_SHEET`, `TAX_INVOICE`, `PROFORMA`, `PAYMENT_VOUCHER`, `BP_CODE`  
3. Legal policy `IN_GST_TAX_INVOICE`  
4. Default concurrency profiles: `GAPLESS_STRICT`, `BUFFERED_STANDARD`  
5. Templates for India bilty + tax invoice  
6. Permission catalog `number_series.*`  
7. Threshold default 80% / remaining 500  

---

## 16. ER overview

```text
document_type ──► series_object ──┬── definitions ── segments / format / check_digit
                                  ├── assignments ── intervals ── counter / buffers
                                  ├── legal_policy bindings
                                  └── document_bindings

assignment ── allocation ledger ── void / recycle
           ── reservations
           ── thresholds
           ── fiscal reset / rollover

changeset ── approval ── publish_version
package ── items
gap_scan ── findings
```

---

## 17. Implementation notes

1. Never update `formatted_number` in place after ISSUED.  
2. Gapless allocator must be single-keyed per `interval_id`.  
3. Buffer leases expire and return unused tail only if mode allows (non-continuous).  
4. `RANDOM_TOKEN` segment forbidden under gapless legal policies.  
5. Store `scope_hash` on every allocation for forensic queries.  
6. Split models: `catalog`, `definition`, `scope`, `counter`, `allocation`, `legal`, `governance`, `ops`, `plumbing`.
