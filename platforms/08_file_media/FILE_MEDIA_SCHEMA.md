# JeslotERP File Media Platform — Production Schema (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `media_object` / `media_blob` HTTP persist on AsyncSession. Scan engine is a port; live ClamAV is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p08_file_media`  
**PostgreSQL schema:** `media`  
**Companion:** [`FILE_MEDIA_GUIDE.md`](FILE_MEDIA_GUIDE.md) · [`FILE_MEDIA_API.md`](FILE_MEDIA_API.md)

> Runtime models: `platforms/p08_file_media/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `media` (never `p08`) |
| Tables | `media_*` |
| Soft delete | Status + `deleted_at`; purge removes blob |
| Cross-schema | UUID refs only |
| RLS | FORCE on tenant-scoped tables |
| Secrets | `secret_ref_key` only — values in configuration |
| Checksums | SHA-256 hex lowercase |
| Object keys | Tenant-prefixed; immutable after AVAILABLE |

---

## 2. Complete table inventory (**62 tables**)

### 2.1 Storage backends & policies (9)

| # | Table | Purpose |
|---|---|---|
| 1 | `media_storage_backend` | S3/Azure/GCS/Local registry |
| 2 | `media_storage_backend_tenant` | Tenant → backend binding |
| 3 | `media_bucket_profile` | Bucket/prefix/SSE profile |
| 4 | `media_content_policy` | MIME/ext/size rules |
| 5 | `media_content_policy_rule` | Allow/deny rules |
| 6 | `media_classification` | PUBLIC_BRAND, KYC, … |
| 7 | `media_encryption_profile` | CMK/SSE refs |
| 8 | `media_cdn_profile` | CDN endpoint + purge API ref |
| 9 | `media_feature_binding` | Feature-flag gates |

### 2.2 Media objects & blobs (8)

| # | Table | Purpose |
|---|---|---|
| 10 | `media_object` | Logical media header |
| 11 | `media_blob` | Physical object pointer |
| 12 | `media_checksum` | Multi-algo checksums |
| 13 | `media_object_tag` | Tags |
| 14 | `media_object_label` | Key/value labels |
| 15 | `media_content_address` | Hash → blob dedupe index |
| 16 | `media_object_alias` | External/legacy ids |
| 17 | `media_metadata_exif` | Scrubbed/retained EXIF JSON |

### 2.3 Upload sessions (7)

| # | Table | Purpose |
|---|---|---|
| 18 | `media_upload_session` | Init session |
| 19 | `media_upload_part` | Multipart parts |
| 20 | `media_upload_signature` | Issued signed URL audit |
| 21 | `media_quota_reservation` | Byte reservations |
| 22 | `media_upload_abort` | Abort reasons |
| 23 | `media_proxy_upload_audit` | Rare proxy uploads |
| 24 | `media_complete_challenge` | Optional verify token |

### 2.4 Scanning & quarantine (6)

| # | Table | Purpose |
|---|---|---|
| 25 | `media_scan_engine` | AV engine registry |
| 26 | `media_scan_job` | Scan job |
| 27 | `media_scan_result` | Findings |
| 28 | `media_quarantine` | Quarantine record |
| 29 | `media_quarantine_review` | Admin decisions |
| 30 | `media_threat_signature` | Optional signature hits |

### 2.5 Variants & processing (7)

| # | Table | Purpose |
|---|---|---|
| 31 | `media_variant_profile` | thumb_sm, img_web, … |
| 32 | `media_variant` | Variant instances |
| 33 | `media_processing_job` | Async jobs |
| 34 | `media_processing_attempt` | Retries |
| 35 | `media_ocr_result` | OCR payload ref / text |
| 36 | `media_preview_page` | PDF page previews |
| 37 | `media_watermark_profile` | Preview watermark rules |

### 2.6 Attachments & collections (6)

| # | Table | Purpose |
|---|---|---|
| 38 | `media_attachment_link` | Polymorphic entity links |
| 39 | `media_attachment_purpose` | POD, KYC_FRONT, … |
| 40 | `media_collection` | Light folder/album |
| 41 | `media_collection_item` | Collection membership |
| 42 | `media_share_grant` | Explicit share to user/role |
| 43 | `media_share_link` | Time-boxed share tokens (hashed) |

### 2.7 Access, ACL, audit (6)

| # | Table | Purpose |
|---|---|---|
| 44 | `media_acl_entry` | Principal ACL on media |
| 45 | `media_download_audit` | Download/preview log |
| 46 | `media_access_token_jti` | Revocable download JTIs |
| 47 | `media_signed_url_policy` | Template constraints |
| 48 | `media_ip_allowlist` | Optional download IP rules |
| 49 | `media_abuse_event` | Rate/abuse signals |

### 2.8 Quota, retention, lifecycle (8)

| # | Table | Purpose |
|---|---|---|
| 50 | `media_quota_policy` | Limits |
| 51 | `media_quota_usage` | Current usage |
| 52 | `media_retention_policy` | Retention rules |
| 53 | `media_legal_hold` | Hold records |
| 54 | `media_lifecycle_policy` | HOT/COOL/ARCHIVE rules |
| 55 | `media_lifecycle_transition` | Transition history |
| 56 | `media_restore_request` | Archive restore |
| 57 | `media_purge_job` | Hard purge runs |

### 2.9 Governance, packs, plumbing (5 + plumbing)

| # | Table | Purpose |
|---|---|---|
| 58 | `media_policy_package` | MIME/retention packs |
| 59 | `media_policy_package_item` | Pack contents |
| 60 | `media_changeset` | Policy change batches |
| 61 | `media_approval` | Approvals for policy |
| 62 | `media_usage_stats` | Aggregates |

**Plumbing (mandatory):** `media_outbox`, `media_idempotency_key`, `media_catalog_audit`

**Implementation total with plumbing: 65 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `media_status` | `INIT`, `UPLOADING`, `UPLOADED`, `SCANNING`, `AVAILABLE`, `QUARANTINED`, `FAILED`, `SOFT_DELETED`, `PURGED`, `ARCHIVED`, `RESTORING` |
| `media_scan_status` | `PENDING`, `CLEAN`, `INFECTED`, `SUSPICIOUS`, `ERROR`, `SKIPPED` |
| `media_backend_kind` | `S3_COMPAT`, `AZURE_BLOB`, `GCS`, `LOCAL_FS`, `SAP_CONTENT_SERVER` |
| `media_tier` | `HOT`, `COOL`, `ARCHIVE` |
| `media_job_type` | `AV_SCAN`, `THUMBNAIL`, `PDF_PREVIEW`, `IMAGE_COMPRESS`, `OCR`, `EXIF_SCRUB`, `CDN_PURGE`, `TIER_MOVE`, `RESTORE`, `PURGE` |
| `media_job_status` | `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELLED` |
| `media_link_visibility` | `INHERIT`, `INTERNAL`, `RESTRICTED`, `PUBLIC_READ` |
| `media_share_status` | `ACTIVE`, `REVOKED`, `EXPIRED` |
| `media_classification_code` | `PUBLIC_BRAND`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED_KYC`, `LEGAL_HOLD_CONTENT` |

---

## 4. Backends & policies (detail)

### 4.1 `media_storage_backend`

| Column | Type | Notes |
|---|---|---|
| `backend_key` | VARCHAR(50) UNIQUE | `s3_primary` |
| `kind` | VARCHAR(30) | |
| `endpoint_url` | TEXT NULL | |
| `region` | VARCHAR(50) NULL | |
| `bucket_or_container` | VARCHAR(200) | |
| `path_prefix_template` | VARCHAR(200) | `{env}/{tenant_id}/…` |
| `secret_ref_key` | VARCHAR(150) | configuration secret |
| `sse_mode` | VARCHAR(30) NULL | `AES256`, `AWS_KMS`, … |
| `encryption_profile_id` | UUID NULL | |
| `is_active` | BOOLEAN | |
| `allows_public_cdn` | BOOLEAN | |

### 4.2 `media_content_policy_rule`

| Column | Type | Notes |
|---|---|---|
| `policy_id` | UUID | |
| `rule_type` | VARCHAR(30) | ALLOW_EXT, DENY_MIME, MAX_BYTES, MAX_IMAGE_PX |
| `pattern` | VARCHAR(200) | `pdf`, `image/*` |
| `value_num` | BIGINT NULL | |
| `priority` | INT | |

---

## 5. Media object & blob

### 5.1 `media_object`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | RLS |
| `company_id` | UUID NULL | |
| `status` | VARCHAR(20) | |
| `classification_code` | VARCHAR(40) | |
| `original_filename` | VARCHAR(500) | Sanitized display name |
| `content_type` | VARCHAR(150) | |
| `byte_size` | BIGINT | |
| `checksum_sha256` | VARCHAR(64) NULL | |
| `storage_backend_id` | UUID | |
| `owner_user_id` | UUID NULL | |
| `created_by` | UUID NULL | |
| `deleted_at` | TIMESTAMPTZ NULL | |
| `legal_hold_flag` | BOOLEAN DEFAULT false | Denorm |
| `tier` | VARCHAR(20) DEFAULT 'HOT' | |
| `dedupe_of_media_id` | UUID NULL | If aliased to canonical |
| `label_key` | VARCHAR(200) NULL | Optional i18n |

### 5.2 `media_blob`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | Parent logical (original) OR variant owner |
| `variant_key` | VARCHAR(50) DEFAULT 'original' | |
| `object_key` | TEXT NOT NULL | Storage key |
| `etag` | VARCHAR(200) NULL | Provider etag |
| `byte_size` | BIGINT | |
| `content_type` | VARCHAR(150) | |
| `checksum_sha256` | VARCHAR(64) | |
| `is_primary` | BOOLEAN | |
| `storage_class` | VARCHAR(30) NULL | STANDARD/GLACIER… |

**Unique:** `(media_id, variant_key)` active.

### 5.3 `media_content_address`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | Dedupe scoped per tenant |
| `checksum_sha256` | VARCHAR(64) | |
| `canonical_blob_id` | UUID | |
| `ref_count` | INT | |

---

## 6. Upload session

### 6.1 `media_upload_session`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | |
| `tenant_id` | UUID | |
| `upload_mode` | VARCHAR(20) | SINGLE, MULTIPART, PROXY |
| `status` | VARCHAR(20) | |
| `expected_size` | BIGINT NULL | |
| `expected_content_type` | VARCHAR(150) | |
| `multipart_upload_id` | VARCHAR(200) NULL | Provider id |
| `part_size` | INT NULL | |
| `expires_at` | TIMESTAMPTZ | |
| `initiated_by` | UUID | |
| `idempotency_key` | VARCHAR(100) NULL | |

### 6.2 `media_upload_part`

| Column | Type | Notes |
|---|---|---|
| `session_id` | UUID | |
| `part_number` | INT | |
| `etag` | VARCHAR(200) NULL | |
| `checksum_sha256` | VARCHAR(64) NULL | |
| `byte_size` | BIGINT NULL | |
| `status` | VARCHAR(20) | |

---

## 7. Scan & quarantine

Scan engine is a `ScanPort`, not a vendor client baked into HTTP. `media_scan_engine` is the registry row (`stub_clam` seed). Pytest/dev uses `MemoryScanEngine`. `ClamAvScanEngine` never invents CLEAN/INFECTED — ping/scan are `PROVIDER_PENDING` until a real daemon + INSTREAM exist. Unknown/ERROR scan status fail-closes the object to FAILED.

### 7.1 `media_scan_job`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | |
| `engine_id` | UUID | |
| `status` | VARCHAR(20) | |
| `started_at` / `finished_at` | TIMESTAMPTZ | |
| `attempts` | INT | |

### 7.2 `media_quarantine`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID UNIQUE | |
| `reason_code` | VARCHAR(50) | MALWARE, POLICY, MAGIC_MISMATCH |
| `detail` | JSONB | |
| `quarantined_at` | TIMESTAMPTZ | |
| `reviewed_at` | TIMESTAMPTZ NULL | |
| `resolution` | VARCHAR(30) NULL | RELEASE, DESTROY, KEEP |

---

## 8. Variants & jobs

### 8.1 `media_variant_profile`

| Column | Type | Notes |
|---|---|---|
| `profile_key` | VARCHAR(50) | `thumb_md` |
| `applies_to_mime` | VARCHAR(100) | `image/*`, `application/pdf` |
| `params` | JSONB | width/height/quality/page |
| `is_active` | BOOLEAN | |

### 8.2 `media_processing_job`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | |
| `job_type` | VARCHAR(30) | |
| `status` | VARCHAR(20) | |
| `priority` | INT | |
| `payload` | JSONB | |
| `result_variant_key` | VARCHAR(50) NULL | |
| `error_code` | VARCHAR(50) NULL | |

---

## 9. Attachments & shares

### 9.1 `media_attachment_link`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `media_id` | UUID | |
| `entity_type` | VARCHAR(100) | `bp.kyc_case`, `sales.order` |
| `entity_id` | UUID | |
| `purpose_code` | VARCHAR(50) | `POD`, `KYC_AADHAAR` |
| `visibility` | VARCHAR(20) | |
| `sort_order` | INT | |
| `linked_by` | UUID | |
| `is_primary` | BOOLEAN | |

**Unique live:** `(tenant_id, entity_type, entity_id, purpose_code, media_id)`.

### 9.2 `media_share_link`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | |
| `token_hash` | VARCHAR(64) | Store hash only |
| `expires_at` | TIMESTAMPTZ | |
| `max_downloads` | INT NULL | |
| `download_count` | INT | |
| `password_hash` | VARCHAR(100) NULL | Optional |
| `status` | VARCHAR(20) | |
| `created_by` | UUID | |

---

## 10. Quota, retention, lifecycle

### 10.1 `media_quota_policy`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | Null = system default |
| `scope_type` | VARCHAR(20) | TENANT/COMPANY/USER/CLASSIFICATION |
| `scope_id` | UUID NULL | |
| `max_bytes` | BIGINT | |
| `max_objects` | BIGINT NULL | |
| `max_upload_bytes` | BIGINT NULL | Per object |

### 10.2 `media_legal_hold`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | |
| `hold_key` | VARCHAR(100) | Case/ref |
| `reason` | TEXT | |
| `applied_by` | UUID | |
| `released_at` | TIMESTAMPTZ NULL | |
| `is_active` | BOOLEAN | |

### 10.3 `media_restore_request`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | |
| `status` | VARCHAR(20) | |
| `requested_by` | UUID | |
| `restored_tier` | VARCHAR(20) | Usually HOT |
| `expires_at` | TIMESTAMPTZ NULL | Temp restore window |

---

## 11. Access audit

### 11.1 `media_download_audit`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | |
| `tenant_id` | UUID | |
| `user_id` | UUID NULL | |
| `access_kind` | VARCHAR(20) | DOWNLOAD, PREVIEW, SIGNED_ISSUE |
| `variant_key` | VARCHAR(50) | |
| `ip` | INET NULL | |
| `user_agent` | TEXT NULL | |
| `granted` | BOOLEAN | |
| `deny_reason` | VARCHAR(50) NULL | |
| `created_at` | TIMESTAMPTZ | |

---

## 12. Packages & governance

- `media_policy_package` — e.g. `india.kyc.media@1.0.0` (MIME + retention for KYC)  
- Changeset/approval for tightening deny rules in production  
- `media_usage_stats` — daily bytes/objects per tenant  

---

## 13. Plumbing

| Table | Purpose |
|---|---|
| `media_outbox` | Domain events |
| `media_idempotency_key` | Init/complete/delete idempotency |
| `media_catalog_audit` | Policy/backend audit JSON |

---

## 14. RLS summary

| Class | Policy |
|---|---|
| Backends/profiles system | Read limited; admin manage |
| Media objects, blobs, sessions, links | FORCE `tenant_id` |
| Quarantine reviews | FORCE + `media.admin` |
| Share links | FORCE tenant; token lookup via hashed API carefully |

---

## 15. Seed minimum

1. Backends: `local_dev`, `s3_primary` (config-driven)  
2. Classifications list  
3. Content policy `default_erp` — allow pdf/png/jpeg/webp/docx/xlsx; deny exe/bat/js  
4. Variant profiles: `thumb_sm`, `thumb_md`, `img_web`, `preview_pdf_page1`  
5. Purposes: `POD`, `KYC_FRONT`, `KYC_BACK`, `INVOICE_PDF`, `VEHICLE_RC`, `DRIVER_LICENSE`, `BRAND_LOGO`, `GENERIC`  
6. Quota defaults (tenant)  
7. Retention defaults (soft delete 30d → purge)  
8. Permissions `media.*`  
9. Scan engine placeholder row  

---

## 16. ER overview

```text
storage_backend ── bucket_profile ── encryption / cdn
content_policy ── rules
media_object ── blob(s) / checksums / tags
     │
     ├── upload_session ── parts / signatures / quota_reservation
     ├── scan_job ── results → quarantine
     ├── processing_job → variants / ocr / previews
     ├── attachment_link / collection / share
     ├── acl / download_audit
     └── legal_hold / lifecycle / purge
```

---

## 17. Implementation notes

1. Never return `secret_ref_key` values resolved in responses.  
2. Complete upload must verify size/checksum against storage HEAD when possible.  
3. Dedupe increments `ref_count`; purge only when zero and not held.  
4. Share tokens stored hashed (SHA-256 of secret).  
5. Quarantine objects may move to isolated prefix/bucket.  
6. Split models: `storage`, `object`, `upload`, `scan`, `variant`, `link`, `security`, `lifecycle`, `plumbing`.
