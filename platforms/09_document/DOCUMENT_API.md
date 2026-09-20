# JeslotERP Document Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — document/version/library list/get Postgres-first; empty list is `[]`. Not ArchiveLink. Not Production.  
**Package:** `platforms.p09_document`  
**PostgreSQL schema:** `document`  
**Public base:** `/api/v1/documents`  
**Internal base:** `/internal/v1/documents`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`DOCUMENT_GUIDE.md`](DOCUMENT_GUIDE.md) · [`DOCUMENT_SCHEMA.md`](DOCUMENT_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | DIR CRUD, versions/media bind, check-out/in, status network, object links, libraries, templates/generate, renditions, ACL, distribution, legal hold, e-sign, compounds, packs. |
| 1.1 | 2026-09-12 | TASK-SOR-011: DIR/versions/libraries durable; empty catalog is `[]`. |

---

## 1. Design principles (advanced)

1. **DIR-first** — create document record; bind content as versions via `media_id`.  
2. **No raw bytes** — upload through p08; pass `media_id` into check-in/create version.  
3. **Status network enforced** — only allowed transitions.  
4. **RELEASED immutability** — content changes require new version + policy.  
5. **Check-out exclusivity** — second check-out → conflict.  
6. **Idempotent create/generate/check-in** — `Idempotency-Key`.  
7. **Number series** — server allocates when type policy requires (client cannot forge).  
8. **ACL on every read/content URL** — fail closed.  
9. **Reverse link query** — by entity is a primary API.  
10. **Controlled copy audited** — distribution always logged.  
11. **Legal hold blocks delete/obsolete**.  
12. **Internal callbacks** for rendition/e-sign provider webhooks.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
If-Match: <version>
X-Tenant-Id: <uuid>
X-Company-Id: <uuid>
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
DOC_NOT_FOUND / VERSION_NOT_FOUND / TYPE_NOT_FOUND
DOC_NUMBER_CONFLICT / ALLOCATION_FAILED
DOC_STATUS_INVALID / TRANSITION_DENIED
DOC_CHECKED_OUT / NOT_CHECKED_OUT / CHECKOUT_EXPIRED / CHECKOUT_OWNED_BY_OTHER
DOC_IMMUTABLE / RELEASE_FORBIDDEN
DOC_MEDIA_REQUIRED / MEDIA_NOT_AVAILABLE / MEDIA_QUARANTINED
DOC_LINK_EXISTS / LINK_NOT_ALLOWED / PRIMARY_LINK_CONFLICT
DOC_ACL_DENIED
DOC_LEGAL_HOLD_ACTIVE / DELETE_FORBIDDEN
DOC_TEMPLATE_NOT_FOUND / MERGE_FIELD_INVALID / GENERATE_FAILED
DOC_RENDITION_NOT_READY / DISTRIBUTION_DENIED
DOC_ESIGN_DENIED / ESIGN_PROVIDER_ERROR
DOC_COMPOUND_INCOMPLETE
DOC_LIBRARY_TYPE_DENIED
DOC_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
DOC_PACKAGE_CHECKSUM_MISMATCH
```

HTTP: `404` · `409` · `422` · `403` · `412`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `document.read` | Read if ACL |
| `document.create` | Create DIR |
| `document.checkout` | Check-out/in/undo |
| `document.release` | Release transitions |
| `document.obsolete` | Obsolete |
| `document.delete` | Soft delete |
| `document.acl.manage` | ACL |
| `document.template.manage` | Templates |
| `document.template.generate` | Generate |
| `document.admin` | Types, break-lock |
| `document.legal_hold` | Holds |
| `document.distribute` | Controlled copies |
| `document.esign` | E-sign |
| `document.audit.read` | Audit |
| `document.*` | All |

---

## 6. Document (DIR) APIs

### 6.1 Create

```http
POST /api/v1/documents
Idempotency-Key: …
```

```json
{
  "type_key": "CONTRACT",
  "title": "Freight MSA — Acme Logistics",
  "company_id": "…",
  "library_id": "…",
  "folder_id": "…",
  "classification_code": "CONFIDENTIAL",
  "media_id": "…",
  "change_comment": "Initial upload",
  "primary_link": {
    "entity_type": "bp.partner",
    "entity_id": "…",
    "link_role": "PRIMARY"
  },
  "labels": { "contract_year": "2026" }
}
```

**Server:** allocates number via p07 when required; creates v1.0 DRAFT (or type default); binds media if provided and AVAILABLE.

**Response:** document summary + `document_number` + `current_version`.

### 6.2 Get / list / patch

```http
GET   /api/v1/documents/{document_id}
GET   /api/v1/documents?type_key=CONTRACT&status=RELEASED&q=Acme
PATCH /api/v1/documents/{document_id}
```

Patch: title, labels, folder move, classification (policy permitting) — not status/content.

### 6.3 Soft delete

```http
POST /api/v1/documents/{document_id}/delete
```

Fails on legal hold / active checkout (unless force admin).

### 6.4 By number

```http
GET /api/v1/documents/by-number/{document_number}
```

---

## 7. Versions & content

### 7.1 List / get version

```http
GET /api/v1/documents/{document_id}/versions
GET /api/v1/documents/{document_id}/versions/{version_id}
```

### 7.2 Add version (without checkout — only if policy allows drafts)

```http
POST /api/v1/documents/{document_id}/versions
Idempotency-Key: …
```

```json
{
  "media_id": "…",
  "bump": "MINOR",
  "change_comment": "Clause 4.2 update",
  "files": [
    { "slot_key": "ANNEX_A", "media_id": "…" }
  ]
}
```

Rejects if RELEASED immutability requires checkout flow.

### 7.3 Content access (signed URL via media gateway)

```http
POST /api/v1/documents/{document_id}/versions/{version_id}/download-url
POST /api/v1/documents/{document_id}/versions/{version_id}/preview-url
```

Checks doc ACL then calls p08; audits `doc_access_audit`.

### 7.4 Compare

```http
POST /api/v1/documents/{document_id}/versions/compare
```

```json
{ "left_version_id": "…", "right_version_id": "…" }
```

Returns job id / diff metadata (engine-specific).

---

## 8. Check-out / check-in

```http
POST /api/v1/documents/{document_id}/checkout
POST /api/v1/documents/{document_id}/checkin
Idempotency-Key: …
POST /api/v1/documents/{document_id}/checkout/undo
POST /api/v1/documents/{document_id}/checkout/break   # document.admin
GET  /api/v1/documents/{document_id}/checkout
```

**Checkout body (optional):** `{ "ttl_seconds": 3600, "client_info": "WebApp" }`

**Check-in body:**

```json
{
  "media_id": "…",
  "bump": "MAJOR",
  "change_comment": "Executed copy",
  "transition_to": "IN_REVIEW"
}
```

Creates new version, clears lock, optional status transition.

---

## 9. Status network

```http
GET  /api/v1/documents/{document_id}/transitions
POST /api/v1/documents/{document_id}/transitions
GET  /api/v1/documents/{document_id}/status-history
```

**Transition body:**

```json
{
  "to_status": "RELEASED",
  "comment": "Legal approved",
  "process_instance_id": "optional-p10"
}
```

Special endpoints (sugar):

```http
POST /api/v1/documents/{document_id}/release
POST /api/v1/documents/{document_id}/obsolete
POST /api/v1/documents/{document_id}/submit-review
```

On RELEASED: mark current version immutable; emit `document.released`; enqueue default renditions.

---

## 10. Object links

```http
POST   /api/v1/documents/{document_id}/links
GET    /api/v1/documents/{document_id}/links
DELETE /api/v1/documents/{document_id}/links/{link_id}

GET    /api/v1/documents/by-entity
```

**By entity (primary reverse lookup):**

```http
GET /api/v1/documents/by-entity?entity_type=sales.order&entity_id=…&link_role=EVIDENCE
```

**Create link:**

```json
{
  "entity_type": "sales.order",
  "entity_id": "…",
  "link_role": "EVIDENCE",
  "is_primary": false
}
```

---

## 11. Libraries & folders

```http
GET    /api/v1/documents/cabinets
GET    /api/v1/documents/libraries
POST   /api/v1/documents/libraries
GET    /api/v1/documents/libraries/{id}/folders
POST   /api/v1/documents/folders
PATCH  /api/v1/documents/folders/{id}
POST   /api/v1/documents/{document_id}/move
```

**Move:** `{ "library_id": "…", "folder_id": "…" }`

---

## 12. Templates & generate

```http
GET    /api/v1/documents/templates
POST   /api/v1/documents/templates
GET    /api/v1/documents/templates/{template_key}
PUT    /api/v1/documents/templates/{template_key}/fields
POST   /api/v1/documents/templates/{template_key}/versions   # media_id of template file

POST   /api/v1/documents/generate
Idempotency-Key: …
GET    /api/v1/documents/generate/{job_id}
```

**Generate body:**

```json
{
  "template_key": "freight.contract.v1",
  "entity_type": "bp.partner",
  "entity_id": "…",
  "company_id": "…",
  "field_overrides": {
    "effective_date": "2026-09-09"
  },
  "auto_link_primary": true
}
```

Flow: resolve merge fields → render → p08 store → create DIR+version → optional link.

---

## 13. Renditions & controlled distribution

```http
POST /api/v1/documents/{document_id}/renditions
GET  /api/v1/documents/{document_id}/versions/{version_id}/renditions
GET  /api/v1/documents/renditions/{rendition_id}

POST /api/v1/documents/{document_id}/distribute
GET  /api/v1/documents/{document_id}/distributions
```

**Rendition request:** `{ "kind": "PDF_WATERMARKED", "profile_key": "PDF_CONTROLLED" }`

**Distribute:**

```json
{
  "version_id": "…",
  "kind": "PDF_WATERMARKED",
  "recipients": [
    { "label": "Acme Legal", "email": "legal@acme.example" }
  ],
  "purpose": "COUNTERPARTY_COPY",
  "expires_at": "2026-12-31T00:00:00Z"
}
```

Creates controlled copy records + optional notification hook.

---

## 14. ACL & sharing

```http
GET /api/v1/documents/{document_id}/acl
PUT /api/v1/documents/{document_id}/acl
POST /api/v1/documents/{document_id}/shares
POST /api/v1/documents/shares/{grant_id}/revoke
```

**ACL put:**

```json
{
  "entries": [
    { "principal_type": "ROLE", "principal_id": "…", "perm": "VIEW", "grant_or_deny": "GRANT" },
    { "principal_type": "USER", "principal_id": "…", "perm": "CHECKOUT", "grant_or_deny": "GRANT" }
  ]
}
```

---

## 15. Comments & review tasks

```http
GET  /api/v1/documents/{document_id}/comments
POST /api/v1/documents/{document_id}/comments
GET  /api/v1/documents/{document_id}/review-tasks
POST /api/v1/documents/{document_id}/review-tasks
POST /api/v1/documents/review-tasks/{id}/complete
```

Lightweight collaboration; heavy BPM stays in p10.

---

## 16. Legal hold & retention

```http
POST /api/v1/documents/{document_id}/legal-holds
POST /api/v1/documents/legal-holds/{hold_id}/release
GET  /api/v1/documents/{document_id}/legal-holds

GET  /api/v1/documents/retention-policies
PUT  /api/v1/documents/retention-policies/{policy_key}/bindings
```

Applying hold: sets flag + gateway `media.legal_hold` for all version media ids.

---

## 17. E-sign

```http
POST /api/v1/documents/{document_id}/esign/envelopes
GET  /api/v1/documents/esign/envelopes/{envelope_id}
POST /api/v1/documents/esign/envelopes/{envelope_id}/void
POST /api/v1/documents/esign/envelopes/{envelope_id}/remind
POST /internal/v1/documents/esign/webhooks/{provider_key}
```

**Create envelope:**

```json
{
  "version_id": "…",
  "provider_key": "dummy_or_adobe",
  "signers": [
    { "name": "Asha", "email": "asha@acme.example", "order": 1 }
  ]
}
```

On completion: optional new version with signed media + transition.

---

## 18. Compound documents

```http
GET  /api/v1/documents/{document_id}/compound
PUT  /api/v1/documents/{document_id}/compound
POST /api/v1/documents/{document_id}/compound/validate-release
```

**Put body:** ordered child document ids + roles.  
Validate-release ensures required children are RELEASED.

---

## 19. Catalog admin (types / status / packages)

```http
GET  /api/v1/documents/types
POST /api/v1/documents/types
GET  /api/v1/documents/types/{type_key}/policy
PUT  /api/v1/documents/types/{type_key}/policy
GET  /api/v1/documents/types/{type_key}/status-network
PUT  /api/v1/documents/types/{type_key}/status-network

GET  /api/v1/documents/packages
POST /api/v1/documents/packages/{package_key}/install
```

---

## 20. Audit

```http
GET /api/v1/documents/{document_id}/audit/access
GET /api/v1/documents/audit/catalog?entity_type=DOC_TYPE
```

Requires `document.audit.read`.

---

## 21. Internal APIs

| Endpoint | Purpose |
|---|---|
| `POST /internal/v1/documents/renditions/callback` | Worker finished rendition |
| `POST /internal/v1/documents/generate/callback` | Generate worker result |
| `POST /internal/v1/documents/esign/webhooks/{provider}` | Provider events |
| `POST /internal/v1/documents/link` | Trusted domain auto-link |
| `GET  /internal/v1/documents/by-entity` | Service reverse lookup |
| `POST /internal/v1/documents/retention/run` | Scheduled retention |
| `GET  /internal/v1/documents/health` | Readiness |

---

## 22. Caching & concurrency

| Resource | Strategy |
|---|---|
| DIR metadata | ETag / If-Match on patch |
| Check-out | DB unique ACTIVE lock |
| Status transition | Conditional on current status |
| Generate / check-in | Idempotency keys |
| Download URLs | Not shared across users; short TTL |

---

## 23. Example client flows

### 23.1 Upload scanned POD as DMS evidence on bilty

1. p08 init/complete → `media_id` AVAILABLE  
2. `POST /documents` type `POD_SCAN` + media_id + link to bilty `EVIDENCE`  
3. Transition to RELEASED if policy simple  
4. UI lists via `GET /documents/by-entity?entity_type=sales.order&…`

### 23.2 Contract edit cycle

1. Create CONTRACT (number from p07)  
2. `checkout` → edit file → p08 new media  
3. `checkin` with bump MAJOR → IN_REVIEW  
4. Reviewer `transitions` → RELEASED  
5. `distribute` watermarked copy to counterparty  

### 23.3 Generate rate card PDF

1. `POST /generate` template + partner entity  
2. Poll job → open output document  
3. Auto primary link to partner  

### 23.4 KYC pack compound

1. Create parent `KYC_PACK`  
2. Create child KYC item docs; `PUT compound`  
3. `validate-release` before pack RELEASED  

---

## 24. Event hooks

| Event | Consumer |
|---|---|
| `document.released` | Search index; domain UI |
| `document.checked_out` | Collaboration presence |
| `document.link.added` | Entity activity feed |
| `document.generated` | Notify requester |
| `document.distributed` | Notification channel |
| `document.esign.completed` | Finance/legal unlock |
| `document.legal_hold.applied` | Compliance |

---

## 25. Compatibility notes

- Public prefix `/api/v1/documents`; schema `document`.  
- Always use p08 for bytes; p09 never accepts multipart file bodies for content (except optional tiny template metadata).  
- Domain modules should **link** documents, not reinvent version tables.  
- `document_number` display uses p06 labels for type names only.  

---

## 26. Related documents

- Guide: [`DOCUMENT_GUIDE.md`](DOCUMENT_GUIDE.md)  
- Schema: [`DOCUMENT_SCHEMA.md`](DOCUMENT_SCHEMA.md)  
- Media: [`../08_file_media/FILE_MEDIA_API.md`](../08_file_media/FILE_MEDIA_API.md)  
- Number series: [`../07_number_series/NUMBER_SERIES_API.md`](../07_number_series/NUMBER_SERIES_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
