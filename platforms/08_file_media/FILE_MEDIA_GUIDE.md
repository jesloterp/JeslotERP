# JeslotERP File Media Platform — Developer Integration Guide

**Version:** 1.4  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — object/quarantine ledger Postgres-first; ClamAV INSTREAM and S3/Azure/GCS adapters are live protocol when creds exist; pytest / empty creds stay `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p08_file_media`  
**PostgreSQL schema:** `media`  
**Depends on:** `p01_identity`, `p02_organization`  
**Integrates with:** `p03_configuration` (quotas/backends), `p06_localization` (error labels), `p09_document` (DMS consumer), `p10_process`, `p12_feature`, `p14_messaging` (scan/transcode workers), `p18_search` (index hooks), `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`FILE_MEDIA_SCHEMA.md`](FILE_MEDIA_SCHEMA.md) · [`FILE_MEDIA_API.md`](FILE_MEDIA_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise media control plane: multi-backend storage, signed upload/download, multipart, virus scan, variants, dedupe, retention/legal hold, quotas, encryption refs, CDN, attachment links, lifecycle tiers, AV/OCR hooks. |
| 1.1 | 2026-09-12 | TASK-SOR-010: durable blob metadata; empty list is `[]`; S3 port `PROVIDER_PENDING`; RLS on `require_media_access`. |
| 1.2 | 2026-09-12 | TASK-SOR-023: ClamAV `ScanPort` fail-closed; pytest never invents live CLEAN/INFECTED; `POST /scan-engines/{key}/test-connection`. |
| 1.3 | 2026-09-12 | HYG-018 pointer to TASK-SOR-023. Stay in p08. No INSTREAM client. |
| 1.4 | 2026-09-12 | PROD-LIVE-008: ClamAV `zINSTREAM` + S3/Azure/GCS SDK adapters. Pytest never invents CLEAN or a live bucket. |

---

## 1. Purpose (enterprise)

`p08_file_media` is JeslotERP’s **binary content & media control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP** — Content Server / ArchiveLink / KPro (store, retrieve, content repositories)  
- **Microsoft Dynamics 365 + Azure Blob / SharePoint** — attachments, file columns, SAS URLs  
- **Salesforce** — ContentVersion / ContentDocument / Files, CDN distribution  
- **Modern object storage platforms** — S3/Azure/GCS with lifecycle, encryption, multipart  

It is **not** “save file to `/uploads`”. It is the system that makes ERP attachments production-safe for:

1. **POD photos, LR scans, invoices, KYC docs, vehicle RC, driver licenses**  
2. **Direct-to-storage uploads** via short-lived signed URLs (no API proxy for GB files)  
3. **Multipart** resumable uploads for large PDFs/videos  
4. **Malware / AV scanning** before content becomes downloadable  
5. **Content-type allowlists** and magic-byte verification  
6. **Variants** — thumbnails, PDF page previews, compressed derivatives  
7. **Content-hash deduplication** (optional per tenant policy)  
8. **Quotas** — tenant / company / user / bucket class  
9. **Retention, legal hold, soft-delete, hard purge**  
10. **Encryption-at-rest key refs**, CDN purge, download audit, lifecycle tiers (HOT/COOL/ARCHIVE)

### Owns

| Domain | Examples |
|---|---|
| Media objects | Logical file identity, metadata, status |
| Blobs / storage | Backend refs, keys, checksums, sizes |
| Upload sessions | Init, parts, complete, abort |
| Signed access | Upload/download/preview URLs |
| Security scanning | AV status, quarantine |
| Variants / derivatives | Thumb, preview, transcode jobs |
| Attachment links | Polymorphic bind to domain entities |
| Collections (light) | Folders/albums — not full DMS |
| Policies | MIME, size, retention, quota |
| Lifecycle | Tiering, purge jobs |
| Observability | Download audit, scan metrics |

### Does **not** own

| Concern | Owner |
|---|---|
| Document versioning / check-in / templates | `p09_document` |
| Business “Invoice PDF is version 3” semantics | Domain + p09 |
| User auth | `p01_identity` |
| Org masters | `p02_organization` |
| Secret storage for cloud keys | `p03_configuration` secrets |
| Full-text search index | `p18_search` (consumes events) |
| Email sending of attachments | `p15_notification` |
| Long-running worker infra | `p14_messaging` |

### Critical split: Media vs Document

| | **File Media (p08)** | **Document (p09)** |
|---|---|---|
| Stores | Bytes + object metadata + variants | DMS doc, versions, check-out, templates |
| Question | Where is the blob and is it safe? | What business document is this? |
| Identity | `media_id` | `document_id` → references `media_id`s |

**Rule:** Domain modules attach `media_id`. DMS wraps media into versioned documents when needed.

---

## 2. Architectural position

```text
Client / Mobile / Scanner
        │
        ▼
   signed URL (PUT) ──────────────────► Object Storage (S3/Azure/GCS/Local)
        │                                      │
        │ complete                             │ event
        ▼                                      ▼
   p08 media API ◄──── AV / variant workers ──┘
        │
        ├──► p09 document (optional DMS)
        ├──► BP KYC / Bilty POD / Invoice PDF
        └──► search / audit / notify
```

**Hard rules**

1. API process should **not** stream multi-GB bodies in production — prefer signed direct upload.  
2. No cross-schema FKs — UUID refs only.  
3. Quarantined media is **never** downloadable except security roles.  
4. RLS fail-closed on tenant media.  
5. Storage credentials live in configuration secrets; media stores `secret_ref_key` / backend id only.  
6. Soft-deleted media remains until purge policy; legal hold blocks purge.

---

## 3. Advanced design principles

1. **Logical media ≠ physical blob** — one media may reference blob + many variants.  
2. **Content-addressable option** — SHA-256 dedupe when policy allows.  
3. **Status machine** — `INIT` → `UPLOADING` → `UPLOADED` → `SCANNING` → `AVAILABLE` \| `QUARANTINED` → `DELETED` → `PURGED`.  
4. **Defense in depth** — extension allowlist + MIME + magic bytes + AV.  
5. **Least privilege URLs** — method, path, content-type, max size, expiry bound in signature.  
6. **Variant pipeline** — async jobs; originals immutable once AVAILABLE.  
7. **Attachment links** — many entities can reference one media (with ACL checks at link time).  
8. **Quota reservation** — reserve bytes on init; release on abort/fail.  
9. **Lifecycle tiers** — HOT → COOL → ARCHIVE with restore workflow.  
10. **Encryption refs** — CMK/KEK identifiers, never raw keys in DB.  
11. **PII / EXIF scrub** — policy for images (strip GPS).  
12. **Download audit** — who fetched what, when, from where.  
13. **CDN invalidation** — on delete/replace for public-ish assets (logos).  
14. **Idempotent complete** — same upload session complete is safe.  
15. **CQRS HTTP** — thin routers; storage adapter gateway.  
16. **Multipart first-class** — > threshold uses multipart automatically.  
17. **Preview ≠ download** — separate permission / watermark hook.  
18. **Tenant isolation** — storage key prefix includes tenant_id.

---

## 4. Core concepts

### 4.1 Media object

Logical file:

```text
media_id, tenant_id, original_filename, content_type,
byte_size, checksum_sha256, status, classification,
created_by, storage_backend_id, object_key
```

### 4.2 Upload flows

**A. Direct signed (preferred)**

```text
1. POST /uploads/init → media_id + upload_url(s) + headers
2. Client PUT to storage
3. POST /uploads/{id}/complete → scan enqueued
4. Poll or webhook → AVAILABLE
```

**B. Multipart**

```text
init → N part URLs → complete multipart → scan
```

**C. Proxy upload (dev / small only)**

```text
POST /uploads/proxy  (size capped, feature-flagged)
```

### 4.3 Status machine

```text
INIT
  → UPLOADING
  → UPLOADED
  → SCANNING
       ├─► AVAILABLE
       ├─► QUARANTINED (malware / policy)
       └─► FAILED
AVAILABLE → SOFT_DELETED → PURGED
AVAILABLE → LEGAL_HOLD (flag; blocks delete/purge)
ARCHIVE tier: AVAILABLE_ARCHIVED → RESTORING → AVAILABLE
```

### 4.4 Variants

| Variant key | Use |
|---|---|
| `original` | Source bytes |
| `thumb_sm` / `thumb_md` | Lists / cards |
| `preview_pdf_page1` | Document UI |
| `img_web` | Compressed web image |
| `ocr_text` | Derived text artifact (optional media) |

Variants are child media or `media_variant` rows pointing at blob keys.

### 4.5 Attachment link

Polymorphic association:

```text
media_id + entity_type + entity_id + purpose (POD, KYC_FRONT, INVOICE_PDF)
+ visibility + linked_by
```

Does not grant storage ACL by itself — download still checks media ACL + link + caller permissions.

### 4.6 Classification

`PUBLIC_BRAND`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED_KYC`, `LEGAL_HOLD_CONTENT`

Influences CDN eligibility, download permission, retention defaults.

---

## 5. Storage backends

| Backend kind | Notes |
|---|---|
| `S3_COMPAT` | AWS S3 / MinIO |
| `AZURE_BLOB` | Azure |
| `GCS` | Google |
| `LOCAL_FS` | Dev only; forbidden in prod profile |
| `SAP_CONTENT_SERVER` | Optional adapter later |

Each backend: endpoint, region, bucket, path prefix pattern, `secret_ref_key`, SSE mode.

**Key layout:**

```text
{env}/{tenant_id}/{yyyy}/{mm}/{media_id}/original
{env}/{tenant_id}/{yyyy}/{mm}/{media_id}/variants/{variant_key}
```

---

## 6. Security

### 6.1 Scanning

- AV engine is a `ScanPort`. Pytest / `MEDIA_SCAN_PROVIDER=dev` uses `MemoryScanEngine` (EICAR → INFECTED).  
- Live ClamAV attaches only when `MEDIA_SCAN_PROVIDER` is not a stub, `CLAMAV_HOST` is set, **and** ping succeeds. Pytest never attaches.  
- Without a daemon, `ClamAvScanEngine` returns `ERROR` / `PROVIDER_PENDING` — never invented CLEAN or INFECTED. Upload fail-closes to FAILED. Live scan uses ClamAV `zINSTREAM` only when `CLAMAV_HOST` is set and pytest is not running.  
- `scan_status`: PENDING / CLEAN / INFECTED / ERROR.  
- Infected → QUARANTINED; admin review; no download for normal roles.

### 6.2 Content validation

1. Allowed extensions per policy profile  
2. Declared Content-Type  
3. Magic-byte sniff  
4. Max size  
5. Image dimension limits  
6. PDF javascript / encrypted PDF policy flags  

### 6.3 Access

| Permission | Use |
|---|---|
| `media.read` | Metadata + download if ACL allows |
| `media.upload` | Init/complete uploads |
| `media.delete` | Soft delete |
| `media.purge` | Hard purge (ops) |
| `media.admin` | Quarantine review, backend manage |
| `media.legal_hold` | Apply/remove hold |
| `media.quota.manage` | Quotas |
| `media.policy.manage` | MIME/retention policies |
| `media.audit.read` | Download/scan audit |
| `media.*` | Wildcard |

Signed download URLs embed principal + expiry; still logged.

### 6.4 RLS

FORCE RLS on `tenant_id` for media, sessions, links, quotas.  
System backends readable; secrets never returned.

---

## 7. Quotas & reservation

```text
quota scopes: TENANT | COMPANY | USER | CLASSIFICATION
metrics: bytes_used, object_count
on init: reserve expected_size
on complete: finalize actual_size
on abort/fail/delete: release
```

Exceed → `MEDIA_QUOTA_EXCEEDED`.

---

## 8. Retention & legal hold

- `retention_policy`: days keep after soft delete / after create  
- `legal_hold` flag blocks soft delete & purge  
- Purge worker skips holds; emits `media.purge.skipped`  
- KYC docs often longer retention via policy pack  

---

## 9. Lifecycle tiers

| Tier | Meaning |
|---|---|
| HOT | Immediate download |
| COOL | Cheaper; slight latency OK |
| ARCHIVE | Requires restore job before download |

`POST /media/{id}/restore` → async → AVAILABLE from ARCHIVE.

---

## 10. Integration rules

1. Store **`media_id`** on domain rows — never raw storage URLs in business tables.  
2. p09 Document versions reference media ids for each version blob.  
3. i18n screenshots / BP KYC / bilty POD all use p08.  
4. Generate signed URLs server-side only.  
5. Workers consume outbox / queue for scan & variants.  
6. Gateways only — no ORM cross-imports.  
7. Configuration holds max sizes & backend selection per tenant.

---

## 11. Module layout

```text
platforms/p08_file_media/
  application/
    services/
      upload_session.py
      signed_url.py
      complete_upload.py
      scan_orchestrator.py
      variant_pipeline.py
      quota_service.py
      retention.py
      lifecycle.py
      dedupe.py
      magic_sniff.py
      access_guard.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/…
    persistence/…
    storage/  # s3.py azure.py gcs.py local.py
    messaging/outbox/
    workers/scan_consumer.py variant_consumer.py purge_consumer.py
  tests/unit/upload/ scan/ signed_url/
```

---

## 12. Domain events

| Event | When |
|---|---|
| `media.upload.initialized` / `completed` / `aborted` | Upload lifecycle |
| `media.scan.clean` / `infected` / `failed` | AV |
| `media.available` | Ready for download |
| `media.variant.ready` | Derivative done |
| `media.deleted` / `purged` | Removal |
| `media.legal_hold.applied` / `released` | Hold |
| `media.tier.changed` / `restore.completed` | Lifecycle |
| `media.quota.breached` | Quota |
| `media.download.audited` | Optional high-signal |

Stream: `jesloterp:media:outbox`.

---

## 13. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, backends, permissions |
| P2 | Init/complete signed upload + local/S3 adapter |
| P3 | Multipart + checksum verify |
| P4 | AV scan + quarantine |
| P5 | Variants (thumb/pdf preview) |
| P6 | Quotas + retention + soft delete/purge |
| P7 | Attachment links + download audit |
| P8 | Lifecycle tiers + legal hold + dedupe |
| P9 | CDN purge + policy packs |
| P10 | Registry → **Live** |

---

## 14. Definition of Done (enterprise)

- [x] Signed upload works against S3-compatible backend *(LOCAL_FS test double; S3_COMPAT is `PROVIDER_PENDING` without credentials)*  
- [x] Tenant RLS GUCs on HTTP (`require_media_access`)  
- [x] Object/quarantine persist on `AsyncSession` (empty catalog is `[]`)  
- [x] Multipart complete verifies part checksums  
- [x] Infected file never returns download URL for normal user  
- [x] Live ClamAV is a fail-closed port (`PROVIDER_PENDING`); pytest stays MEMORY  
- [x] Magic-byte mismatch rejected  
- [x] Quota reservation rolls back on abort  
- [x] Legal hold blocks purge  
- [x] Tenant RLS on media + links *(ORM ready; FORCE RLS Alembic deferred to parent)*  
- [x] Variant job idempotent  
- [x] Soft delete hides from default lists  
- [x] No storage secrets in API responses  
- [x] No cross-schema FKs  

---

## 15. Anti-patterns

| Don’t | Do |
|---|---|
| Store files on API local disk in prod | Object storage backend |
| Put permanent public S3 URLs in DB | Store media_id; sign on demand |
| Skip AV “for speed” | Scan before AVAILABLE |
| Trust client Content-Type alone | Magic sniff + allowlist |
| Delete blob rows without audit | Soft delete → purge job |
| Let p09 re-implement upload | p09 calls p08 |
| Embed base64 GB files in JSON | Signed direct PUT |

---

## 16. Related documents

- Schema: [`FILE_MEDIA_SCHEMA.md`](FILE_MEDIA_SCHEMA.md)  
- API: [`FILE_MEDIA_API.md`](FILE_MEDIA_API.md)  
- Document DMS: [`../09_document/DOCUMENT_GUIDE.md`](../09_document/DOCUMENT_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
