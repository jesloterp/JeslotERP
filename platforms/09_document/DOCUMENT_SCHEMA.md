# JeslotERP Document Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — DIR/version/library HTTP persist on AsyncSession. Not Production.  
**Package:** `platforms.p09_document`  
**PostgreSQL schema:** `document`  
**Companion:** [`DOCUMENT_GUIDE.md`](DOCUMENT_GUIDE.md) · [`DOCUMENT_API.md`](DOCUMENT_API.md)

> Runtime models: `platforms/p09_document/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `document` (never `p09`) |
| Tables | `doc_*` |
| Content | **UUID `media_id` only** — no bytea for files |
| Soft delete | `deleted_at` + status; versions retained |
| Cross-schema | UUID refs (media, number allocation, org, users) |
| RLS | FORCE on tenant DIRs and children |
| Released content | Immutable version rows |

---

## 2. Complete table inventory (**64 tables**)

### 2.1 Types, status, classification (10)

| # | Table | Purpose |
|---|---|---|
| 1 | `doc_type` | Document types (CONTRACT, POD_SCAN, …) |
| 2 | `doc_type_policy` | Checkout required, numbering, etc. |
| 3 | `doc_status` | Status codes catalog |
| 4 | `doc_status_network` | Network header per type |
| 5 | `doc_status_transition` | Allowed edges + permissions |
| 6 | `doc_classification` | Confidentiality levels |
| 7 | `doc_category` | Business categories |
| 8 | `doc_category_type` | M2M category↔type |
| 9 | `doc_series_binding` | type → number_series object_key |
| 10 | `doc_feature_binding` | Feature flags |

### 2.2 Libraries & structure (6)

| # | Table | Purpose |
|---|---|---|
| 11 | `doc_cabinet` | Top-level cabinet |
| 12 | `doc_library` | Library |
| 13 | `doc_folder` | Folder tree |
| 14 | `doc_folder_closure` | Closure table for tree queries |
| 15 | `doc_library_type_allow` | Allowed types per library |
| 16 | `doc_library_default_acl` | Default ACL templates |

### 2.3 Document info records (8)

| # | Table | Purpose |
|---|---|---|
| 17 | `doc_document` | DIR header |
| 18 | `doc_document_alias` | External/legacy numbers |
| 19 | `doc_document_tag` | Tags |
| 20 | `doc_document_label` | Key/value labels |
| 21 | `doc_document_party` | Related BP roles on doc |
| 22 | `doc_document_relation` | Doc-to-doc relations |
| 23 | `doc_description` | Long description / abstract |
| 24 | `doc_language` | Doc language code (i18n ref) |

### 2.4 Versions & content refs (8)

| # | Table | Purpose |
|---|---|---|
| 25 | `doc_version` | Version header → media_id |
| 26 | `doc_version_file` | Additional files on a version |
| 27 | `doc_version_attribute` | Snapshot attributes |
| 28 | `doc_rendition` | Derived renditions |
| 29 | `doc_rendition_profile` | PDF/watermark profiles |
| 30 | `doc_content_role` | Role catalog |
| 31 | `doc_checksum_mirror` | Optional mirrored checksum from media |
| 32 | `doc_version_compare` | Stored compare jobs/results meta |

### 2.5 Check-out & collaboration (5)

| # | Table | Purpose |
|---|---|---|
| 33 | `doc_checkout` | Active check-out locks |
| 34 | `doc_checkout_history` | Lock history |
| 35 | `doc_break_lock` | Admin break-lock audit |
| 36 | `doc_comment` | Review comments |
| 37 | `doc_review_task` | Review assignments |

### 2.6 Object links (4)

| # | Table | Purpose |
|---|---|---|
| 38 | `doc_object_link` | Polymorphic entity links |
| 39 | `doc_link_role` | PRIMARY, EVIDENCE, … |
| 40 | `doc_link_history` | Link add/remove audit |
| 41 | `doc_entity_type_allow` | Allowed entity types per doc type |

### 2.7 Templates & generation (6)

| # | Table | Purpose |
|---|---|---|
| 42 | `doc_template` | Template definitions |
| 43 | `doc_template_version` | Template content media_id |
| 44 | `doc_merge_field` | Allow-listed merge fields |
| 45 | `doc_merge_binding` | Field → source path |
| 46 | `doc_generate_job` | Generate runs |
| 47 | `doc_generate_result` | Output doc/version ids |

### 2.8 ACL, share, distribution (6)

| # | Table | Purpose |
|---|---|---|
| 48 | `doc_acl_entry` | Princip ACL |
| 49 | `doc_share_grant` | Explicit grants |
| 50 | `doc_distribution` | Controlled distribution header |
| 51 | `doc_distribution_recipient` | Recipients |
| 52 | `doc_controlled_copy` | Watermarked copy instances |
| 53 | `doc_access_audit` | View/download/print audit |

### 2.9 Retention, hold, e-sign, compound (7)

| # | Table | Purpose |
|---|---|---|
| 54 | `doc_retention_policy` | Retention schedules |
| 55 | `doc_retention_binding` | Bind to type/library |
| 56 | `doc_legal_hold` | Document holds |
| 57 | `doc_esign_provider` | Provider registry (secret ref) |
| 58 | `doc_esign_envelope` | Envelope state |
| 59 | `doc_esign_signer` | Signers |
| 60 | `doc_compound_node` | Parent/child structure |

### 2.10 Governance & ops (4 + plumbing)

| # | Table | Purpose |
|---|---|---|
| 61 | `doc_changeset` | Type/policy changes |
| 62 | `doc_approval` | Approvals |
| 63 | `doc_package` | Type/template packs |
| 64 | `doc_package_item` | Pack items |

**Plumbing:** `doc_outbox`, `doc_idempotency_key`, `doc_catalog_audit`, `doc_status_history`

**Implementation total with plumbing: ~68 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `doc_lifecycle_status` | `DRAFT`, `IN_REVIEW`, `REJECTED`, `APPROVED`, `RELEASED`, `OBSOLETE`, `DELETED` |
| `doc_checkout_status` | `ACTIVE`, `RELEASED`, `EXPIRED`, `BROKEN` |
| `doc_version_bump` | `MINOR`, `MAJOR`, `PATCH` |
| `doc_link_role_code` | `PRIMARY`, `SUPPORTING`, `EVIDENCE`, `SIGNED_COPY`, `ANNEX`, `SUPERSEDES` |
| `doc_rendition_kind` | `PDF`, `PDF_WATERMARKED`, `PREVIEW`, `TEXT_EXTRACT` |
| `doc_acl_perm` | `VIEW`, `EDIT`, `CHECKOUT`, `RELEASE`, `DELETE`, `ADMIN`, `DISTRIBUTE` |
| `doc_esign_status` | `DRAFT`, `SENT`, `VIEWED`, `SIGNED`, `DECLINED`, `EXPIRED`, `VOIDED` |
| `doc_generate_status` | `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED` |
| `doc_relation_type` | `SUPERSEDES`, `AMENDS`, `REFERENCES`, `TRANSLATION_OF`, `PART_OF` |

---

## 4. Types & status (detail)

### 4.1 `doc_type`

| Column | Type | Notes |
|---|---|---|
| `type_key` | VARCHAR(50) UNIQUE | `CONTRACT`, `POD_SCAN`, `KYC_PACK` |
| `name` | VARCHAR(150) | |
| `label_key` | VARCHAR(200) NULL | i18n |
| `default_classification` | VARCHAR(40) | |
| `is_system` | BOOLEAN | |
| `is_active` | BOOLEAN | |

### 4.2 `doc_type_policy`

| Column | Type | Notes |
|---|---|---|
| `type_id` | UUID | |
| `require_checkout` | BOOLEAN | |
| `require_number_series` | BOOLEAN | |
| `series_object_key` | VARCHAR(50) NULL | p07 key |
| `allow_multiple_primary_links` | BOOLEAN | |
| `immutable_on_release` | BOOLEAN DEFAULT true | |
| `bump_major_on_release` | BOOLEAN | |
| `require_approval_to_release` | BOOLEAN | |
| `process_key` | VARCHAR(100) NULL | p10 process |
| `default_retention_policy_id` | UUID NULL | |
| `allow_esign` | BOOLEAN | |
| `max_versions` | INT NULL | Soft cap |

### 4.3 `doc_status_transition`

| Column | Type | Notes |
|---|---|---|
| `network_id` | UUID | |
| `from_status` | VARCHAR(30) | |
| `to_status` | VARCHAR(30) | |
| `permission_code` | VARCHAR(80) NULL | |
| `require_process` | BOOLEAN | |
| `require_comment` | BOOLEAN | |

---

## 5. DIR & versions

### 5.1 `doc_document`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | RLS |
| `company_id` | UUID NULL | |
| `document_number` | VARCHAR(100) | From p07 or external |
| `allocation_id` | UUID NULL | p07 allocation ref |
| `type_id` | UUID | |
| `title` | VARCHAR(300) | |
| `status_code` | VARCHAR(30) | |
| `classification_code` | VARCHAR(40) | |
| `library_id` | UUID NULL | |
| `folder_id` | UUID NULL | |
| `current_version_id` | UUID NULL | |
| `owner_user_id` | UUID NULL | |
| `checked_out_by` | UUID NULL | Denorm |
| `legal_hold_flag` | BOOLEAN DEFAULT false | |
| `released_at` | TIMESTAMPTZ NULL | |
| `obsolete_at` | TIMESTAMPTZ NULL | |
| `deleted_at` | TIMESTAMPTZ NULL | |
| `primary_entity_type` | VARCHAR(100) NULL | Hint |
| `primary_entity_id` | UUID NULL | Hint |

**Unique live:** `(tenant_id, document_number)` when number present.

### 5.2 `doc_version`

| Column | Type | Notes |
|---|---|---|
| `document_id` | UUID | |
| `version_label` | VARCHAR(20) | `1.0` |
| `revision` | INT | Monotonic |
| `major` / `minor` / `patch` | INT | |
| `media_id` | UUID NULL | p08 — required if contentful |
| `content_type` | VARCHAR(150) NULL | Denorm |
| `byte_size` | BIGINT NULL | Denorm |
| `checksum_sha256` | VARCHAR(64) NULL | Denorm |
| `change_comment` | TEXT NULL | |
| `created_by` | UUID | |
| `status_at_create` | VARCHAR(30) | |
| `is_immutable` | BOOLEAN | Set true on release |
| `supersedes_version_id` | UUID NULL | |

**Unique:** `(document_id, revision)`.

### 5.3 `doc_version_file`

Extra slots on a version (annex files): `media_id`, `slot_key`, `sort_order`.

### 5.4 `doc_rendition`

| Column | Type | Notes |
|---|---|---|
| `version_id` | UUID | |
| `kind` | VARCHAR(30) | |
| `profile_id` | UUID NULL | |
| `media_id` | UUID | Output |
| `status` | VARCHAR(20) | |
| `created_at` | TIMESTAMPTZ | |

---

## 6. Check-out

### 6.1 `doc_checkout`

| Column | Type | Notes |
|---|---|---|
| `document_id` | UUID UNIQUE active | One active lock |
| `version_id` | UUID | Base version |
| `checked_out_by` | UUID | |
| `checked_out_at` | TIMESTAMPTZ | |
| `expires_at` | TIMESTAMPTZ NULL | |
| `client_info` | VARCHAR(200) NULL | |
| `status` | VARCHAR(20) | ACTIVE |

Partial unique: one ACTIVE checkout per document.

---

## 7. Object links

### 7.1 `doc_object_link`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `document_id` | UUID | |
| `entity_type` | VARCHAR(100) | `sales.order`, `bp.partner` |
| `entity_id` | UUID | |
| `link_role` | VARCHAR(30) | |
| `is_primary` | BOOLEAN | |
| `linked_by` | UUID | |
| `valid_from` / `valid_to` | TIMESTAMPTZ NULL | |

**Unique live:** `(document_id, entity_type, entity_id, link_role)`.

Indexes: `(tenant_id, entity_type, entity_id)` for reverse lookup.

---

## 8. Templates

### 8.1 `doc_template`

| Column | Type | Notes |
|---|---|---|
| `template_key` | VARCHAR(100) UNIQUE | `freight.contract.v1` |
| `type_id` | UUID | Output doc type |
| `name` | VARCHAR(150) | |
| `engine` | VARCHAR(30) | `DOCX_MERGE`, `HTML_PDF` |
| `is_active` | BOOLEAN | |

### 8.2 `doc_merge_field`

| Column | Type | Notes |
|---|---|---|
| `template_id` | UUID | |
| `field_key` | VARCHAR(100) | `partner.name` |
| `value_type` | VARCHAR(20) | STRING/DATE/MONEY/… |
| `source_path` | VARCHAR(200) | Gateway path allow-list |
| `is_required` | BOOLEAN | |

### 8.3 `doc_generate_job`

| Column | Type | Notes |
|---|---|---|
| `template_id` | UUID | |
| `entity_type` / `entity_id` | | Context |
| `status` | VARCHAR(20) | |
| `idempotency_key` | VARCHAR(100) | |
| `output_document_id` | UUID NULL | |
| `output_version_id` | UUID NULL | |
| `output_media_id` | UUID NULL | |
| `error_code` | VARCHAR(50) NULL | |

---

## 9. ACL & distribution

### 9.1 `doc_acl_entry`

| Column | Type | Notes |
|---|---|---|
| `document_id` | UUID | |
| `principal_type` | VARCHAR(20) | USER/ROLE/GROUP |
| `principal_id` | UUID | |
| `perm` | VARCHAR(20) | |
| `grant_or_deny` | VARCHAR(10) | GRANT/DENY |

### 9.2 `doc_controlled_copy`

| Column | Type | Notes |
|---|---|---|
| `document_id` | UUID | |
| `version_id` | UUID | |
| `rendition_id` | UUID | Watermarked |
| `recipient_label` | VARCHAR(200) | |
| `purpose` | VARCHAR(100) | |
| `expires_at` | TIMESTAMPTZ NULL | |
| `distributed_by` | UUID | |

---

## 10. Retention, hold, e-sign, compound

### 10.1 `doc_legal_hold`

Same pattern as media: `hold_key`, `reason`, `applied_by`, `released_at`, `is_active`.  
On apply: gateway to hold all version `media_id`s.

### 10.2 `doc_esign_envelope`

| Column | Type | Notes |
|---|---|---|
| `document_id` / `version_id` | UUID | |
| `provider_key` | VARCHAR(50) | |
| `external_envelope_id` | VARCHAR(200) NULL | |
| `status` | VARCHAR(20) | |
| `sent_at` / `completed_at` | TIMESTAMPTZ | |

Provider secrets via `secret_ref_key` on `doc_esign_provider`.

### 10.3 `doc_compound_node`

| Column | Type | Notes |
|---|---|---|
| `parent_document_id` | UUID | |
| `child_document_id` | UUID | |
| `position` | INT | |
| `node_role` | VARCHAR(50) | CHAPTER, ANNEX, EVIDENCE |
| `is_required_for_release` | BOOLEAN | |

---

## 11. Libraries

`doc_cabinet` → `doc_library` → `doc_folder` with `parent_folder_id` + `doc_folder_closure` for efficient subtree queries.  
Library binds default retention, ACL, allowed types.

---

## 12. Governance & packs

- Changesets for type/status network edits  
- Packages: `india.kyc.document@1.0.0`, `freight.contract.pack@1.0.0`  
- Seed types, networks, merge fields  

---

## 13. Plumbing

| Table | Purpose |
|---|---|
| `doc_outbox` | Events |
| `doc_idempotency_key` | Create/generate/check-in |
| `doc_catalog_audit` | Type/policy audit |
| `doc_status_history` | Every status transition |

---

## 14. RLS summary

| Class | Policy |
|---|---|
| Types/status seeds | Readable; admin manage |
| Documents, versions, links, checkouts, ACL | FORCE `tenant_id` |
| Templates system | Readable; manage permission |
| E-sign envelopes | FORCE tenant |

---

## 15. Seed minimum

1. Types: `CONTRACT`, `RATE_CARD`, `POD_SCAN`, `LR_SCAN`, `KYC_PACK`, `KYC_ITEM`, `POLICY`, `GENERIC`  
2. Status network for CONTRACT and KYC_PACK  
3. Link roles + content roles  
4. Series bindings → p07 objects (`DOC_CONTRACT`, …)  
5. Library `DEFAULT` cabinet  
6. Merge field catalog sample for freight contract  
7. Rendition profile `PDF_STANDARD`, `PDF_CONTROLLED`  
8. Permissions `document.*`  
9. Retention defaults  

---

## 16. ER overview

```text
doc_type ── policy / status_network / series_binding
cabinet ── library ── folder ── document
document ── versions ── media_id (p08)
         ── checkout
         ── object_links
         ── acl / distribution / controlled_copy
         ── legal_hold / esign / compound_nodes
template ── merge_fields ── generate_jobs
```

---

## 17. Implementation notes

1. On RELEASED: set `doc_version.is_immutable=true`; reject content replace.  
2. Check-in always creates new `doc_version` row.  
3. Denorm content_type/size/checksum from media at bind time.  
4. Delete document ≠ purge media automatically — policy gateway.  
5. Split models: `catalog`, `library`, `document`, `version`, `checkout`, `link`, `template`, `security`, `retention`, `esign`, `plumbing`.
