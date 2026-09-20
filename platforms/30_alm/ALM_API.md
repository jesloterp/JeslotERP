# JeslotERP ALM Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live**  
**Package:** `platforms.p30_alm`  
**PostgreSQL schema:** `alm`  
**Public base:** `/api/v1/alm`  
**Internal base:** `/internal/v1/alm`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`ALM_GUIDE.md`](ALM_GUIDE.md) · [`ALM_SCHEMA.md`](ALM_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Package, export, import, promote, rollback. |

---

## 1. Design principles (advanced)

1. **Seal before export.**  
2. **Import idempotent** on package_key+version.  
3. **PROD promote requires signature.**  
4. **Dependency validation.**

---

## 2. Common headers

Standard + `Idempotency-Key`.

---

## 3. Envelope

Standard.

---

## 4. Errors

```text
ALM_PACKAGE_NOT_FOUND / ALM_NOT_SEALED / ALM_CHECKSUM_MISMATCH
ALM_DEPENDENCY_MISSING / ALM_SIGNATURE_REQUIRED / ALM_PROMOTE_INVALID
ALM_ALREADY_IMPORTED / ALM_FORBIDDEN / ALM_PROVIDER_PENDING / ALM_DEPLOY_FAILED
```

---

## 5. Permissions

See ALM_GUIDE §5.

---

## 6. Environments & packages

```http
GET  /api/v1/alm/environments
GET  /api/v1/alm/packages
POST /api/v1/alm/packages
POST /api/v1/alm/packages/{id}/artifacts
POST /api/v1/alm/packages/{id}/seal
POST /api/v1/alm/packages/{id}/sign
POST /api/v1/alm/agents/{env}/test-connection
```

`POST /sign` calls p28 HSM. Pytest / empty `HSM_PKCS11_LIB` → `503 ALM_PROVIDER_PENDING` (never invents `SHA256-HMAC-LOCAL`).

`POST /agents/{env}/test-connection` pings the deploy agent. Empty `ALM_DEPLOY_{ENV}_URL` / pytest → `PROVIDER_PENDING`.

### 6.1 Create package

```json
{ "package_key": "kernel.sample", "version": "1.0.0", "layer": "SYSTEM_CORE", "depends_on": [] }
```

### 6.2 Add artifact

```json
{ "kind": "METADATA", "artifact_ref": "…", "layer": "SYSTEM_CORE" }
```

---

## 7. Export / import

```http
POST /api/v1/alm/packages/{id}/export
POST /api/v1/alm/imports
```

Export returns `{ package, manifest, artifacts, checksum_sha256 }`.  
Import of same version returns existing package (`ALM_ALREADY_IMPORTED` is 200 idempotent, not 409).  
Checksum mismatch → `409 ALM_CHECKSUM_MISMATCH`.  
Missing dep → `422 ALM_DEPENDENCY_MISSING`.

---

## 8. Promote / rollback

```http
POST /api/v1/alm/packages/{id}/promote
POST /api/v1/alm/packages/{id}/rollback
```

```json
{ "from_env": "QA", "to_env": "PROD" }
```

PROD without an HSM signature → `403 ALM_SIGNATURE_REQUIRED`. Planted `SHA256-HMAC-LOCAL` is rejected. Promote then calls the deploy agent; pytest / empty target URL → `503 ALM_PROVIDER_PENDING` and status is not `PROMOTED`.

Rollback body: `{ "reason": "bad determine" }` → records `previous_version`.

---

## 9. Security matrix

| Case | Result |
|---|---|
| Unauthenticated | 401 |
| No `alm.promote` | 403 |
| Wrong tenant overlay | 404 |

---

## 10. Implementation checklist

- [x] Idempotent import  
- [x] Checksum  
- [x] Signature gate for PROD (HSM only)  
- [x] Rollback metadata  
- [x] Deploy agent + HSM sign fail-closed  

---

## 11. Boundary reminders

Not Git. No secret values in export JSON.

---

## 12. Related documents

- Guide: [`ALM_GUIDE.md`](ALM_GUIDE.md)  
- Schema: [`ALM_SCHEMA.md`](ALM_SCHEMA.md)
