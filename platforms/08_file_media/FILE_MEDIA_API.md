# JeslotERP File Media Platform — Complete API Specification (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — object list/get Postgres-first; S3 and ClamAV `test-connection` are `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p08_file_media`  
**PostgreSQL schema:** `media`  
**Public base:** `/api/v1/media`  
**Internal base:** `/internal/v1/media`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`FILE_MEDIA_GUIDE.md`](FILE_MEDIA_GUIDE.md) · [`FILE_MEDIA_SCHEMA.md`](FILE_MEDIA_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Signed/multipart upload, scan/quarantine, variants, attachments, shares, quotas, retention/legal hold, lifecycle, CDN, audit, internal worker APIs. |
| 1.1 | 2026-09-12 | TASK-SOR-010: durable blob metadata; S3 test-connection `PROVIDER_PENDING`. |
| 1.2 | 2026-09-12 | TASK-SOR-023: `POST /scan-engines/{engine_key}/test-connection`; ClamAV never invents CLEAN. |

---

## 1. Design principles (advanced)

1. **Direct-to-storage first** — browsers/mobile PUT via signed URLs; API orchestrates.  
2. **AVAILABLE only after scan** — download URLs denied until CLEAN (unless policy skip for trusted internal).  
3. **media_id is the contract** — never persist permanent storage URLs in domain tables.  
4. **Idempotent init/complete/delete** — `Idempotency-Key` on mutating upload ops.  
5. **Least-privilege signatures** — bound method, key, content-type, max-size, expiry.  
6. **Variant async** — clients poll or subscribe; originals immutable.  
7. **Quarantine isolation** — infected content inaccessible to normal roles.  
8. **Legal hold wins** — blocks delete/purge.  
9. **Quota reserve on init** — abort releases.  
10. **Download audit** — issue and fetch paths log access.  
11. **Proxy upload capped** — feature-flag + small max only.  
12. **Internal workers** — scan/variant/purge callbacks authenticated separately.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Company-Id: <uuid>
```

For binary proxy (rare):

```http
Content-Type: application/pdf
Content-Length: …
X-Media-Filename: pod-scan.pdf
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {
    "request_id": "…",
    "idempotent_replay": false
  }
}
```

---

## 4. Errors

```text
MEDIA_NOT_FOUND / BLOB_NOT_FOUND / SESSION_NOT_FOUND
MEDIA_STATUS_INVALID
MEDIA_UPLOAD_EXPIRED / UPLOAD_INCOMPLETE / PART_MISSING
MEDIA_CHECKSUM_MISMATCH / SIZE_MISMATCH / MAGIC_MISMATCH
MEDIA_CONTENT_TYPE_DENIED / EXTENSION_DENIED / SIZE_EXCEEDED
MEDIA_QUOTA_EXCEEDED / QUOTA_RESERVATION_FAILED
MEDIA_SCAN_PENDING / QUARANTINED / SCAN_FAILED
MEDIA_DOWNLOAD_DENIED / PREVIEW_DENIED / SHARE_DENIED
MEDIA_LEGAL_HOLD_ACTIVE / DELETE_FORBIDDEN / PURGE_FORBIDDEN
MEDIA_TIER_ARCHIVED / RESTORE_IN_PROGRESS / RESTORE_REQUIRED
MEDIA_VARIANT_NOT_READY / VARIANT_UNSUPPORTED
MEDIA_BACKEND_UNAVAILABLE / SIGNATURE_FAILED
MEDIA_SHARE_EXPIRED / SHARE_MAXED / SHARE_REVOKED
MEDIA_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
MEDIA_POLICY_VIOLATION / PACKAGE_CHECKSUM_MISMATCH
MEDIA_CDN_PURGE_FAILED
```

HTTP: `404` · `409` · `422` · `403` · `413` size · `429` abuse · `503` backend.

---

## 5. Permissions

| Code | Use |
|---|---|
| `media.read` | Metadata + download when ACL allows |
| `media.upload` | Init/complete/abort |
| `media.delete` | Soft delete |
| `media.purge` | Hard purge |
| `media.admin` | Backends, quarantine review |
| `media.legal_hold` | Holds |
| `media.quota.manage` | Quotas |
| `media.policy.manage` | Content/retention policies |
| `media.share.manage` | Share links/grants |
| `media.audit.read` | Download/scan audit |
| `media.*` | All |

---

## 6. Upload APIs (primary)

### 6.1 Init single-object signed upload

```http
POST /api/v1/media/uploads/init
Idempotency-Key: …
```

```json
{
  "filename": "pod-front.jpg",
  "content_type": "image/jpeg",
  "byte_size": 1844221,
  "checksum_sha256": "optional-client-hash",
  "classification_code": "INTERNAL",
  "company_id": "…",
  "purpose_hint": "POD"
}
```

**Response:**

```json
{
  "media_id": "…",
  "upload_session_id": "…",
  "upload_mode": "SINGLE",
  "status": "UPLOADING",
  "expires_at": "2026-09-09T04:15:00Z",
  "upload": {
    "method": "PUT",
    "url": "https://storage…/…?X-Amz-Signature=…",
    "headers": {
      "Content-Type": "image/jpeg"
    },
    "max_bytes": 1844221
  }
}
```

### 6.2 Init multipart

```http
POST /api/v1/media/uploads/init-multipart
```

```json
{
  "filename": "lr-batch.pdf",
  "content_type": "application/pdf",
  "byte_size": 52428800,
  "part_size": 8388608
}
```

**Response:** session + `parts: [{ part_number, url, headers }]`.

### 6.3 Complete upload

```http
POST /api/v1/media/uploads/{upload_session_id}/complete
Idempotency-Key: …
```

```json
{
  "checksum_sha256": "…",
  "parts": [
    { "part_number": 1, "etag": "\"…\"" }
  ]
}
```

Transitions: UPLOADED → SCANNING (enqueue).  
Response includes `media_id`, `status`.

### 6.4 Abort

```http
POST /api/v1/media/uploads/{upload_session_id}/abort
```

Releases quota reservation; aborts multipart on provider.

### 6.5 List part status

```http
GET /api/v1/media/uploads/{upload_session_id}
GET /api/v1/media/uploads/{upload_session_id}/parts
```

### 6.6 Proxy upload (capped)

```http
POST /api/v1/media/uploads/proxy
Content-Type: application/pdf
X-Media-Filename: small.pdf
```

Requires feature flag; max size from policy (e.g. 5–10 MB). Returns media_id + scanning status.

---

## 7. Media object APIs

### 7.1 Get / list

```http
GET /api/v1/media/objects/{media_id}
GET /api/v1/media/objects?status=AVAILABLE&q=pod&company_id=…
```

Default lists exclude SOFT_DELETED / PURGED / QUARANTINED (unless `media.admin` + filter).

### 7.2 Patch metadata

```http
PATCH /api/v1/media/objects/{media_id}
```

```json
{
  "original_filename": "pod-trailer-12.jpg",
  "classification_code": "CONFIDENTIAL",
  "labels": { "vehicle_no": "MH12AB1234" }
}
```

Cannot change bytes/status via patch.

### 7.3 Soft delete / restore soft

```http
POST /api/v1/media/objects/{media_id}/delete
POST /api/v1/media/objects/{media_id}/undelete
```

Fails with `MEDIA_LEGAL_HOLD_ACTIVE` when held.

### 7.4 Hard purge

```http
POST /api/v1/media/objects/{media_id}/purge
```

Requires `media.purge`; deletes blobs; status PURGED.

---

## 8. Download & preview

### 8.1 Issue signed download URL

```http
POST /api/v1/media/objects/{media_id}/download-url
```

```json
{
  "variant_key": "original",
  "ttl_seconds": 120
}
```

**Rules:**

- Status must be AVAILABLE (or ARCHIVED → `MEDIA_RESTORE_REQUIRED`)  
- Not QUARANTINED  
- ACL / attachment / share checks  
- Writes `media_download_audit` (`SIGNED_ISSUE`)

**Response:**

```json
{
  "method": "GET",
  "url": "https://…",
  "expires_at": "…",
  "content_type": "image/jpeg",
  "byte_size": 1844221,
  "checksum_sha256": "…"
}
```

### 8.2 Preview URL

```http
POST /api/v1/media/objects/{media_id}/preview-url
```

Prefer `thumb_md` / `preview_pdf_page1`; may apply watermark profile. Separate permission semantics (`preview` vs full download) via ACL flags.

### 8.3 Head / metadata for workers

```http
GET /api/v1/media/objects/{media_id}/blobs/{variant_key}
```

---

## 9. Variants & processing

```http
GET  /api/v1/media/objects/{media_id}/variants
POST /api/v1/media/objects/{media_id}/variants/reprocess
GET  /api/v1/media/processing-jobs/{job_id}
GET  /api/v1/media/objects/{media_id}/ocr
GET  /api/v1/media/objects/{media_id}/preview-pages
```

**Reprocess body:** `{ "profiles": ["thumb_md", "img_web"] }`

---

## 10. Scan & quarantine

```http
GET  /api/v1/media/objects/{media_id}/scan
POST /api/v1/media/objects/{media_id}/scan/requeue          # media.admin
GET  /api/v1/media/quarantine
GET  /api/v1/media/quarantine/{media_id}
POST /api/v1/media/quarantine/{media_id}/reviews
```

**Review body:**

```json
{
  "resolution": "DESTROY",
  "notes": "Confirmed malware"
}
```

`RELEASE` only if false positive and engine override permission; still may force re-scan.

Live ClamAV is a port. Without a daemon, scan/ping are `ERROR` / `PROVIDER_PENDING` and the object fail-closes to FAILED. Pytest uses `MemoryScanEngine` (EICAR → QUARANTINED).

---

## 11. Attachment links & collections

### 11.1 Links

```http
POST   /api/v1/media/attachments
GET    /api/v1/media/attachments?entity_type=sales.order&entity_id=…
DELETE /api/v1/media/attachments/{link_id}
POST   /api/v1/media/attachments/bulk
```

**Create:**

```json
{
  "media_id": "…",
  "entity_type": "sales.order",
  "entity_id": "…",
  "purpose_code": "POD",
  "is_primary": true,
  "visibility": "INHERIT"
}
```

### 11.2 Collections

```http
GET    /api/v1/media/collections
POST   /api/v1/media/collections
POST   /api/v1/media/collections/{id}/items
DELETE /api/v1/media/collections/{id}/items/{media_id}
```

Light folders only — not DMS.

---

## 12. Sharing

```http
POST   /api/v1/media/objects/{media_id}/shares
GET    /api/v1/media/objects/{media_id}/shares
POST   /api/v1/media/shares/{share_id}/revoke
POST   /api/v1/media/share-links/redeem          # public/token path may be separate route
```

**Create share link:**

```json
{
  "ttl_seconds": 86400,
  "max_downloads": 5,
  "password": "optional",
  "variant_key": "original"
}
```

Response returns **one-time plaintext token** + share_id; only hash stored.

**Grants (user/role):**

```http
PUT /api/v1/media/objects/{media_id}/acl
GET /api/v1/media/objects/{media_id}/acl
```

---

## 13. Legal hold, retention, lifecycle

```http
POST /api/v1/media/objects/{media_id}/legal-holds
POST /api/v1/media/legal-holds/{hold_id}/release
GET  /api/v1/media/objects/{media_id}/legal-holds

GET  /api/v1/media/retention-policies
PUT  /api/v1/media/retention-policies/{policy_key}

POST /api/v1/media/objects/{media_id}/tier
POST /api/v1/media/objects/{media_id}/restore
GET  /api/v1/media/restore-requests/{id}
```

**Tier change:** `{ "tier": "COOL" }` — async transition job.  
**Restore:** from ARCHIVE → HOT temporary or permanent per body flag.

---

## 14. Quotas

```http
GET /api/v1/media/quotas/me
GET /api/v1/media/quotas?scope_type=TENANT
PUT /api/v1/media/quotas/policies/{id}
GET /api/v1/media/usage-stats?from=…&to=…
```

---

## 15. Policies, backends, packs (admin)

```http
GET  /api/v1/media/backends
POST /api/v1/media/backends
PATCH /api/v1/media/backends/{backend_key}
POST /api/v1/media/backends/{backend_key}/test-connection

POST /api/v1/media/scan-engines/{engine_key}/test-connection

GET  /api/v1/media/content-policies
PUT  /api/v1/media/content-policies/{policy_key}/rules

GET  /api/v1/media/variant-profiles
PUT  /api/v1/media/variant-profiles/{profile_key}

GET  /api/v1/media/packages
POST /api/v1/media/packages/{package_key}/install
```

S3 `test-connection` is `PROVIDER_PENDING` without credentials (no fake bucket).  
Scan-engine `test-connection`: `stub_clam` / `memory` → MEMORY `OK`; `clamav` / `clamd` / `clam` → `ok: false`, `status: PROVIDER_PENDING` (no invented CLEAN). Requires `media.admin`.

---

## 16. CDN

```http
POST /api/v1/media/objects/{media_id}/cdn-purge
POST /api/v1/media/cdn/purge-prefix
```

For `PUBLIC_BRAND` assets (logos). Requires admin/policy.

---

## 17. Audit

```http
GET /api/v1/media/audit/downloads?media_id=…&from=…&to=…
GET /api/v1/media/audit/catalog?entity_type=CONTENT_POLICY
```

Requires `media.audit.read`.

---

## 18. Internal / worker APIs

| Endpoint | Purpose |
|---|---|
| `POST /internal/v1/media/scan/callback` | Engine result → clean/quarantine |
| `POST /internal/v1/media/processing/callback` | Variant/OCR job result |
| `POST /internal/v1/media/lifecycle/callback` | Tier/restore/purge progress |
| `GET  /internal/v1/media/objects/{id}/storage-locator` | Worker fetch key + backend |
| `POST /internal/v1/media/objects/{id}/mark-available` | Trusted complete path |
| `POST /internal/v1/media/purge/run` | Scheduled purge kick |
| `GET  /internal/v1/media/health` | Backend + scanner sample; `clamav` is `PROVIDER_PENDING` |

Internal auth: `X-Internal-Token` + service identity. Locators never exposed publicly.

---

## 19. Caching & concurrency

| Resource | Strategy |
|---|---|
| Media metadata | Short TTL; invalidate on status change |
| Signed URLs | Not cached shared across users |
| Quota usage | Atomic increment/reserve |
| Complete upload | Idempotent; provider complete once |
| Variant jobs | Unique (media_id, job_type, profile) in-flight |

---

## 20. Example client flows

### 20.1 Mobile POD photo

1. `POST /uploads/init` with image/jpeg + size  
2. PUT bytes to signed URL  
3. `POST …/complete`  
4. Poll `GET /objects/{id}` until AVAILABLE (or wait event)  
5. `POST /attachments` link to bilty + purpose `POD`  
6. UI uses `preview-url` (`thumb_md`)

### 20.2 KYC PDF

1. Init with `classification_code=RESTRICTED_KYC`  
2. Complete → scan  
3. On AVAILABLE, link to BP KYC case  
4. Retention pack keeps longer; download audited  

### 20.3 Invoice PDF from generator

1. Internal service uploads via init/complete (or put via worker)  
2. Domain stores `media_id` on invoice  
3. User download-url on demand  

### 20.4 Malware path

1. Complete → SCANNING → INFECTED → QUARANTINED  
2. `download-url` → `403 MEDIA_QUARANTINED`  
3. Admin review DESTROY → purge  

---

## 21. Event hooks

| Event | Consumer |
|---|---|
| `media.available` | Domain UI refresh; p09 version bind; search index |
| `media.scan.infected` | Security notify |
| `media.variant.ready` | UI thumbnail swap |
| `media.deleted` / `purged` | Drop caches; CDN purge |
| `media.quota.breached` | Admin alert |
| `media.restore.completed` | Enable download |

---

## 22. Compatibility notes

- Public prefix `/api/v1/media`; schema name `media`.  
- Clear split: bytes/security here; DMS versions in p09.  
- Clients must handle SCANNING async — do not assume immediate AVAILABLE.  
- Filename is display metadata only; storage key is server-generated.  
- `LOCAL_FS` backend rejected when `ENV=production`.

---

## 23. Related documents

- Guide: [`FILE_MEDIA_GUIDE.md`](FILE_MEDIA_GUIDE.md)  
- Schema: [`FILE_MEDIA_SCHEMA.md`](FILE_MEDIA_SCHEMA.md)  
- Document: [`../09_document/DOCUMENT_GUIDE.md`](../09_document/DOCUMENT_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
