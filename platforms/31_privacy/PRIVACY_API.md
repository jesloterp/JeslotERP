# JeslotERP Privacy Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live**  
**Package:** `platforms.p31_privacy`  
**PostgreSQL schema:** `privacy`  
**Public base:** `/api/v1/privacy`  
**Internal base:** `/internal/v1/privacy`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`PRIVACY_GUIDE.md`](PRIVACY_GUIDE.md) · [`PRIVACY_SCHEMA.md`](PRIVACY_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Subject, consent, DSR, hold block. |

---

## 1. Design principles (advanced)

1. **No blind delete.**  
2. **Hold blocks erasure.**  
3. **Tenant-scoped ACCESS.**  
4. **Auditable mutations.**

---

## 2. Common headers

Standard + `Idempotency-Key`.

---

## 3. Envelope

Standard.

---

## 4. Errors

```text
PRV_SUBJECT_NOT_FOUND / PRV_PURPOSE_NOT_FOUND / PRV_DSR_NOT_FOUND
PRV_LEGAL_HOLD / PRV_FORBIDDEN / PRV_TENANT_REQUIRED
```

---

## 5. Permissions

See PRIVACY_GUIDE §5.

---

## 6. Subjects / purposes / consent / policy

```http
GET  /api/v1/privacy/subjects
POST /api/v1/privacy/subjects
GET  /api/v1/privacy/purposes
POST /api/v1/privacy/consents
POST /api/v1/privacy/consents/{id}/withdraw
GET  /api/v1/privacy/policies
POST /api/v1/privacy/policies
```

---

## 7. Legal hold refs

```http
POST /api/v1/privacy/holds
POST /api/v1/privacy/holds/{id}/release
```

```json
{ "subject_id": "…", "hold_ref": "…" }
```

`hold_ref` is a p19 UUID — not a foreign key.

---

## 8. DSR

```http
GET  /api/v1/privacy/dsrs
POST /api/v1/privacy/dsrs
POST /api/v1/privacy/dsrs/{id}/advance
GET  /api/v1/privacy/dsrs/{id}/operations
```

```json
{ "subject_id": "…", "kind": "ERASURE" }
```

If an ACTIVE hold exists → DSR created as `BLOCKED` with `block_reason=LEGAL_HOLD` (or advance returns `409 PRV_LEGAL_HOLD`).  
ACCESS returns `{ "index": [ … ] }` — at least the p31 subject locator. p04/p01 add locators only when `owner_refs` resolve to a real owner row (`reveal=False`; no invented hits).  
ERASURE advances named owner adapters. Subject may carry `owner_refs` (`p04` partner id, `p01` user id). p31 never DELETEs those rows; missing ref or mock session → operation `SKIPPED`.

---

## 9. Security matrix

| Case | Result |
|---|---|
| Unauthenticated | 401 |
| Wrong tenant subject | 404 |
| Erase under hold | 409 `PRV_LEGAL_HOLD` |

---

## 10. Implementation checklist

- [x] Hold check  
- [x] Consent withdraw  
- [x] RLS  
- [x] No domain DELETE  
- [x] Owning-platform erase adapters (p04/p01)  
- [x] ACCESS corpus crawlers (index not empty)

---

## 11. Boundary reminders

p19 owns audit storage. p31 orchestrates.

---

## 12. Related documents

- Guide: [`PRIVACY_GUIDE.md`](PRIVACY_GUIDE.md)  
- Schema: [`PRIVACY_SCHEMA.md`](PRIVACY_SCHEMA.md)
