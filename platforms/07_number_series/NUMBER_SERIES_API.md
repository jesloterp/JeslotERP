# JeslotERP Number Series Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-09  
**Status:** Live — public `/api/v1/number-series` + internal `/internal/v1/number-series` implemented  
**Package:** `platforms.p07_number_series`  
**PostgreSQL schema:** `number_series`  
**Public base:** `/api/v1/number-series`  
**Internal base:** `/internal/v1/number-series`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`NUMBER_SERIES_GUIDE.md`](NUMBER_SERIES_GUIDE.md) · [`NUMBER_SERIES_SCHEMA.md`](NUMBER_SERIES_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Allocate/reserve/commit/void, peek/simulate, definitions/segments, assignments, legal policies, buffers, rollover, gaps, packs, external intake, idempotency. |

---

## 1. Design principles (advanced)

1. **Allocate-first for runtime** — domain modules use `/allocate`, `/reserve`, `/commit`; catalog APIs are for admins.  
2. **Idempotent issue** — `Idempotency-Key` mandatory on allocate/reserve/commit/manual.  
3. **Mode honesty** — continuous endpoints refuse buffered profiles; legal policy enforced server-side.  
4. **Peek never mutates** — safe for UI previews.  
5. **Scope explicit** — company/branch/FY passed in body; never inferred from unrelated headers alone without JWT tenant.  
6. **Ledger immutable** — void/recycle are append-only state transitions.  
7. **Definition versioning** — activate via publish; no silent in-place pattern edits on ACTIVE.  
8. **External validate-then-register** — manual numbers go through the same uniqueness ledger.  
9. **Internal hot path** — document engines prefer `/internal/v1/number-series/allocate` with service token.  
10. **ETag / If-Match** on definition and assignment updates.  
11. **Threshold & exhaustion** — clear error codes before silent wraparound (wraparound forbidden).  
12. **Simulate** — dry-run format + next sequence without write.

---

## 2. Common headers & scope context

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
If-Match: <version>
X-Tenant-Id: <uuid>
X-Company-Id: <uuid>
X-Branch-Id: <uuid>
```

### Allocate scope body (canonical)

```json
{
  "object_key": "BILTY",
  "company_id": "…",
  "branch_id": "…",
  "fiscal_year_id": "…",
  "posting_date": "2026-09-09",
  "doc_subtype": null,
  "document_ref_type": "sales.order",
  "document_ref_id": "…",
  "channel": "WEB"
}
```

If both `fiscal_year_id` and `posting_date` sent, FY id wins when assignment requires FY; date drives calendar segments.

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
NS_OBJECT_NOT_FOUND / DEFINITION_NOT_FOUND / ASSIGNMENT_NOT_FOUND
NS_INTERVAL_NOT_FOUND / INTERVAL_EXHAUSTED / INTERVAL_CLOSED
NS_SCOPE_INCOMPLETE / SCOPE_AMBIGUOUS
NS_MODE_VIOLATION / LEGAL_POLICY_VIOLATION
NS_DUPLICATE_NUMBER / COLLISION
NS_IDEMPOTENCY_CONFLICT / IN_FLIGHT
NS_RESERVATION_NOT_FOUND / RESERVATION_EXPIRED / RESERVATION_INVALID_STATE
NS_ALLOCATION_NOT_FOUND / INVALID_STATUS_TRANSITION
NS_VOID_FORBIDDEN / RECYCLE_FORBIDDEN
NS_MANUAL_REQUIRED / MANUAL_NOT_ALLOWED / MANUAL_FORMAT_INVALID / CHECK_DIGIT_INVALID
NS_BUFFER_UNAVAILABLE / GAPLESS_LOCK_TIMEOUT
NS_PUBLISH_CONFLICT / APPROVAL_REQUIRED / COVERAGE_BLOCKED
NS_PACKAGE_CHECKSUM_MISMATCH / ALREADY_INSTALLED
NS_THRESHOLD_BREACH_BLOCKING
NS_VERSION_CONFLICT
NS_SIMULATION_ONLY
```

HTTP: `404` · `409` state/conflict · `422` validation/policy · `403` permission · `412` If-Match · `429` lock contention (optional retry-after).

---

## 5. Permissions

| Code | Use |
|---|---|
| `number_series.catalog.read` | Read objects/definitions |
| `number_series.catalog.manage` | Mutate catalog/templates |
| `number_series.assign.manage` | Assignments / bindings |
| `number_series.allocate` | Peek/reserve/allocate/commit |
| `number_series.allocate.manual` | External/manual |
| `number_series.void` | Void |
| `number_series.recycle` | Recycle |
| `number_series.publish` | Activate definitions |
| `number_series.approve` | Approvals |
| `number_series.pack.install` | Packs |
| `number_series.ops.scan` | Gap/reconcile/rollover triggers |
| `number_series.audit.read` | Ledger/audit |
| `number_series.*` | All |

---

## 6. Runtime allocation APIs (primary)

### 6.1 Peek (non-mutating)

```http
POST /api/v1/number-series/peek
```

```json
{
  "object_key": "BILTY",
  "company_id": "…",
  "branch_id": "…",
  "fiscal_year_id": "…",
  "posting_date": "2026-09-09"
}
```

**Response:**

```json
{
  "object_key": "BILTY",
  "preview_formatted_number": "BL/MH01/2526/NDL/000148",
  "next_sequence_value": 148,
  "assignment_id": "…",
  "definition_version": 3,
  "allocation_mode": "NON_CONTINUOUS_BUFFERED",
  "interval": { "from": 1, "to": 999999, "current": 147, "remaining": 999852 }
}
```

Requires `number_series.allocate` (or read+allocate policy). Does not write.

### 6.2 Allocate (atomic issue)

```http
POST /api/v1/number-series/allocate
Idempotency-Key: bilty-create-…
```

Body = scope + optional:

```json
{
  "object_key": "TAX_INVOICE",
  "company_id": "…",
  "fiscal_year_id": "…",
  "posting_date": "2026-09-09",
  "document_ref_type": "finance.tax_invoice",
  "document_ref_id": "…"
}
```

**Response:**

```json
{
  "allocation_id": "…",
  "formatted_number": "INV/MH01/2526/000042",
  "sequence_value": 42,
  "status": "ISSUED",
  "allocation_mode": "CONTINUOUS_GAPLESS",
  "definition_id": "…",
  "interval_id": "…",
  "issued_at": "2026-09-09T04:00:00Z"
}
```

Replay with same Idempotency-Key returns same payload with `meta.idempotent_replay=true`.

### 6.3 Reserve → Commit / Release

```http
POST /api/v1/number-series/reservations
Idempotency-Key: …

POST /api/v1/number-series/reservations/{reservation_id}/commit
Idempotency-Key: …

POST /api/v1/number-series/reservations/{reservation_id}/release
```

**Reserve response:** reservation_id, expires_at, items[{allocation_id, formatted_number, status:RESERVED}].

Commit transitions to ISSUED and binds `document_ref_*`.  
Release/expiry → EXPIRED; sequence handling per mode (gapless: mark void/gap policy; buffered: may return to pool only if never exposed — default: leave gap in non-continuous).

### 6.4 Batch allocate

```http
POST /api/v1/number-series/allocate-batch
Idempotency-Key: …
```

```json
{
  "object_key": "LOADING_SLIP",
  "company_id": "…",
  "count": 10,
  "document_ref_type": "transport.loading_slip_batch",
  "document_ref_id": "…"
}
```

Max `count` capped by config (e.g. 100). Forbidden when `require_gapless` unless policy explicitly allows batch (usually allow for continuous with single lock span).

### 6.5 Manual / external

```http
POST /api/v1/number-series/allocate-manual
Idempotency-Key: …
```

```json
{
  "object_key": "BILTY",
  "company_id": "…",
  "fiscal_year_id": "…",
  "manual_number": "BL/MH01/2526/NDL/900001",
  "document_ref_id": "…"
}
```

Requires `number_series.allocate.manual`. Validates format, check digit, uniqueness → `source=EXTERNAL`.

### 6.6 Validate number (no write)

```http
POST /api/v1/number-series/validate
```

```json
{
  "object_key": "TAX_INVOICE",
  "company_id": "…",
  "formatted_number": "INV/MH01/2526/000042"
}
```

Returns `{ "valid": true, "normalized": "…", "checks": ["FORMAT","CHECK_DIGIT","UNIQUE"] }`.

### 6.7 Void / recycle

```http
POST /api/v1/number-series/allocations/{allocation_id}/void
POST /api/v1/number-series/allocations/{allocation_id}/recycle
```

**Void body:** `{ "reason_code": "DOC_CANCELLED", "reason_text": "…" }`  
Recycle fails with `NS_RECYCLE_FORBIDDEN` when legal policy `forbid_reuse`.

### 6.8 Get allocation

```http
GET /api/v1/number-series/allocations/{allocation_id}
GET /api/v1/number-series/allocations?object_key=BILTY&formatted_number=…
GET /api/v1/number-series/allocations?document_ref_id=…
```

---

## 7. Simulate (admin)

```http
POST /api/v1/number-series/simulate
```

```json
{
  "definition_id": "…",
  "scope": {
    "company_code": "MH01",
    "branch_code": "NDL",
    "fiscal_year_label": "2526",
    "sequence_value": 148
  }
}
```

Returns formatted preview + segment breakdown. No counter mutation (`NS_SIMULATION_ONLY` if someone expects allocate semantics).

---

## 8. Catalog — objects & definitions

### 8.1 Objects

```http
GET    /api/v1/number-series/objects
POST   /api/v1/number-series/objects
GET    /api/v1/number-series/objects/{object_key}
PATCH  /api/v1/number-series/objects/{object_key}
POST   /api/v1/number-series/objects/{object_key}/retire
```

### 8.2 Definitions & segments

```http
GET    /api/v1/number-series/objects/{object_key}/definitions
POST   /api/v1/number-series/objects/{object_key}/definitions
GET    /api/v1/number-series/definitions/{definition_id}
PATCH  /api/v1/number-series/definitions/{definition_id}     # draft only
PUT    /api/v1/number-series/definitions/{definition_id}/segments
GET    /api/v1/number-series/definitions/{definition_id}/segments
POST   /api/v1/number-series/definitions/{definition_id}/clone
```

**Segments replace body:**

```json
{
  "segments": [
    { "position": 1, "segment_kind": "CONSTANT", "literal_value": "BL" },
    { "position": 2, "segment_kind": "SEPARATOR", "literal_value": "/" },
    { "position": 3, "segment_kind": "COMPANY_CODE", "value_source": "scope.company_code" },
    { "position": 4, "segment_kind": "SEPARATOR", "literal_value": "/" },
    { "position": 5, "segment_kind": "FISCAL_YEAR", "format_spec": "YYYY" },
    { "position": 6, "segment_kind": "SEPARATOR", "literal_value": "/" },
    { "position": 7, "segment_kind": "SEQUENCE", "pad_length": 6 }
  ]
}
```

ACTIVE definitions are immutable; edit via clone → changeset → publish.

### 8.3 Templates

```http
GET    /api/v1/number-series/templates
POST   /api/v1/number-series/templates
POST   /api/v1/number-series/templates/{template_key}/apply
```

`apply` creates a draft definition from template for an object.

---

## 9. Assignments, intervals, profiles

```http
GET    /api/v1/number-series/assignments?object_key=BILTY&company_id=…
POST   /api/v1/number-series/assignments
GET    /api/v1/number-series/assignments/{id}
PATCH  /api/v1/number-series/assignments/{id}
POST   /api/v1/number-series/assignments/{id}/deactivate

GET    /api/v1/number-series/intervals?assignment_id=…
POST   /api/v1/number-series/intervals
POST   /api/v1/number-series/intervals/{id}/extend
GET    /api/v1/number-series/intervals/{id}/counter

GET    /api/v1/number-series/concurrency-profiles
PUT    /api/v1/number-series/concurrency-profiles/{profile_key}
```

**Extend:**

```json
{ "new_to_value": 1999999, "reason": "Volume growth FY26" }
```

May require approval when legal policy binds.

---

## 10. Legal policies & thresholds

```http
GET    /api/v1/number-series/legal-policies
POST   /api/v1/number-series/legal-policies
PUT    /api/v1/number-series/legal-policies/{policy_key}/bindings

GET    /api/v1/number-series/thresholds
POST   /api/v1/number-series/thresholds
GET    /api/v1/number-series/threshold-events?status=OPEN
POST   /api/v1/number-series/threshold-events/{id}/ack
```

---

## 11. Governance — changeset / publish

```http
GET    /api/v1/number-series/changesets
POST   /api/v1/number-series/changesets
POST   /api/v1/number-series/changesets/{id}/submit
POST   /api/v1/number-series/changesets/{id}/approvals
POST   /api/v1/number-series/publish
POST   /api/v1/number-series/publish/rollback
GET    /api/v1/number-series/publish/versions
```

**Publish body:**

```json
{
  "changeset_id": "…",
  "assignment_ids": ["…"],
  "notes": "Activate FY26 bilty pattern"
}
```

Activates definition on assignments; emits `number_series.definition.activated`.

---

## 12. Packages

```http
GET    /api/v1/number-series/packages
GET    /api/v1/number-series/packages/{package_key}
POST   /api/v1/number-series/packages/{package_key}/install
POST   /api/v1/number-series/packages/{package_key}/uninstall
```

**Install:**

```json
{
  "version": "1.0.0",
  "checksum": "sha256:…",
  "apply_to_companies": ["…"]
}
```

India GST pack seeds `TAX_INVOICE` + `IN_GST_TAX_INVOICE` policy + gapless profile.

---

## 13. Fiscal rollover & ops

```http
POST   /api/v1/number-series/rollover/runs
GET    /api/v1/number-series/rollover/runs/{id}
POST   /api/v1/number-series/gap-scans
GET    /api/v1/number-series/gap-scans/{id}/findings
POST   /api/v1/number-series/reconcile/runs
GET    /api/v1/number-series/usage-stats?object_key=BILTY&company_id=…
```

**Rollover:**

```json
{
  "object_key": "TAX_INVOICE",
  "company_id": "…",
  "from_fiscal_year_id": "…",
  "to_fiscal_year_id": "…",
  "dry_run": false
}
```

**Gap scan:** continuous series only by default; reports anomalies vs voids.

---

## 14. Import (legacy)

```http
POST   /api/v1/number-series/imports
GET    /api/v1/number-series/imports/{id}
POST   /api/v1/number-series/imports/{id}/commit
```

Upload/metadata body includes `sync_counter` boolean. Items become `source=IMPORT` allocations.

---

## 15. Manual override workflow (hybrid)

```http
POST   /api/v1/number-series/manual-overrides/requests
POST   /api/v1/number-series/manual-overrides/requests/{id}/grants
```

Grant allows a single subsequent `allocate-manual` tied to request id.

---

## 16. Buffer ops (ops / internal)

```http
GET    /api/v1/number-series/buffers?interval_id=…
POST   /internal/v1/number-series/buffers/lease
POST   /internal/v1/number-series/buffers/{lease_id}/checkpoint
POST   /internal/v1/number-series/buffers/{lease_id}/revoke
```

Not for browser clients; allocator workers only.

---

## 17. Audit

```http
GET /api/v1/number-series/audit?entity_type=ALLOCATION&entity_id=…
GET /api/v1/number-series/audit?entity_type=DEFINITION&from=…&to=…
```

Requires `number_series.audit.read`.

---

## 18. Internal platform APIs

| Endpoint | Consumer |
|---|---|
| `POST /internal/v1/number-series/allocate` | Document post engines |
| `POST /internal/v1/number-series/reserve` | Draft document services |
| `POST /internal/v1/number-series/commit` | Post/finalize |
| `POST /internal/v1/number-series/void` | Cancel handlers |
| `GET  /internal/v1/number-series/allocations/by-document` | Reconcile |
| `POST /internal/v1/number-series/cache/invalidate` | After publish |
| `GET  /internal/v1/number-series/health` | Counter DB + lock health |

Internal calls must pass `tenant_id` explicitly + `X-Internal-Token`.

---

## 19. Caching & concurrency

| Resource | Strategy |
|---|---|
| Assignments / active definitions | Short TTL cache; invalidate on publish |
| Gapless allocate | Row lock / advisory lock; `Retry-After` on timeout |
| Buffered allocate | Instance lease + in-proc counter |
| Idempotency keys | Durable until TTL (e.g. 24–72h) |
| Peek | May use read snapshot; never advance |

---

## 20. Example client flows

### 20.1 Post tax invoice (gapless)

1. Validate party/amounts in finance module  
2. `POST /allocate` object `TAX_INVOICE` + company + FY + Idempotency-Key  
3. Persist invoice with `allocation_id` + `formatted_number`  
4. On cancel: `void` (no recycle)

### 20.2 Draft bilty then save

1. `POST /reservations` on create draft  
2. User edits  
3. On save/post: `commit`; on discard: `release`

### 20.3 Legacy stationery

1. `POST /validate` with printed number  
2. `POST /allocate-manual` with permission  
3. Bind document ref

### 20.4 FY open

1. Ops `POST /rollover/runs` for `TAX_INVOICE` per company  
2. Confirm new intervals `from=1`  
3. Threshold rules copied

---

## 21. Event hooks

| Event | Consumer action |
|---|---|
| `number_series.number.issued` | Search index / audit mirror |
| `number_series.number.voided` | Domain cancel sync |
| `number_series.interval.exhausted` | Page ops / block allocate |
| `number_series.threshold.breached` | Notify admin |
| `number_series.definition.activated` | Drop assignment caches |
| `number_series.gap.detected` | Compliance review |

---

## 22. Compatibility notes

- Public prefix: `/api/v1/number-series` (kebab), schema name `number_series`.  
- Always persist `allocation_id` on documents — formatted strings alone are insufficient for void/reconcile.  
- Clients must not local-increment.  
- `RANDOM_TOKEN` segments rejected when legal policy requires gapless.  
- Wraparound past `to_value` is never silent — `NS_INTERVAL_EXHAUSTED`.

---

## 23. Related documents

- Guide: [`NUMBER_SERIES_GUIDE.md`](NUMBER_SERIES_GUIDE.md)  
- Schema: [`NUMBER_SERIES_SCHEMA.md`](NUMBER_SERIES_SCHEMA.md)  
- Organization: [`../02_organization/ORGANIZATION_GUIDE.md`](../02_organization/ORGANIZATION_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
