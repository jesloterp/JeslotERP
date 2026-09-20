# JeslotERP Security Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live**  
**Package:** `platforms.p28_security`  
**PostgreSQL schema:** `security`  
**Public base:** `/api/v1/security`  
**Internal base:** `/internal/v1/security`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`SECURITY_GUIDE.md`](SECURITY_GUIDE.md) · [`SECURITY_SCHEMA.md`](SECURITY_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Keys, rotate, policy, posture, WAF, events. No secret GET. |

---

## 1. Design principles (advanced)

1. **Metadata only on GET** — never key material.  
2. **Idempotent create/rotate** — `Idempotency-Key`.  
3. **Fail-closed tenant.**  
4. **Honest provider status.**  
5. **Thin routers** — `SecurityService` owns writes.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
```

---

## 3. Envelope

```json
{ "success": true, "data": {}, "error": null, "meta": { "request_id": "…" } }
```

---

## 4. Errors

```text
SEC_KEY_NOT_FOUND / SEC_PROVIDER_UNKNOWN / SEC_PROVIDER_PENDING / SEC_ROTATION_IN_FLIGHT
SEC_POLICY_NOT_FOUND / SEC_WAF_NOT_FOUND
SEC_FORBIDDEN / SEC_TENANT_REQUIRED
```

---

## 5. Permissions

See SECURITY_GUIDE §6. Unauthenticated → 401. Wrong permission → 403.

---

## 6. Keys

### 6.1 List keys

```http
GET /api/v1/security/keys
```

Permission: `security.key.read`. Response: key metadata array (no `wrapped_ref`).

### 6.2 Create key

```http
POST /api/v1/security/keys
```

```json
{ "key_alias": "tenant-data", "purpose": "DATA", "algorithm": "AES-256-GCM", "provider_key": "LOCAL_DEV" }
```

**Response:** `{ "key_id", "key_alias", "current_version", "status", "provider_key" }`.

Unknown provider → `422 SEC_PROVIDER_UNKNOWN`.  
`AWS_KMS` / `AZURE_KV` / `HSM` without live credentials → `503 SEC_PROVIDER_PENDING` (no invented wrap handle).

### 6.3 Get key

```http
GET /api/v1/security/keys/{key_id}
```

Must not include plaintext or `_secret_plain`.

### 6.4 Rotate key

```http
POST /api/v1/security/keys/{key_id}/rotate
```

Permission: `security.key.rotate`. Concurrent → `409 SEC_ROTATION_IN_FLIGHT`.

### 6.5 Retire key

```http
POST /api/v1/security/keys/{key_id}/retire
```

---

## 7. Providers / policy / posture / WAF / events

```http
GET  /api/v1/security/providers
POST /api/v1/security/providers/{provider_key}/test-connection
GET  /api/v1/security/policies
POST /api/v1/security/policies
GET  /api/v1/security/posture
POST /api/v1/security/posture/run
GET  /api/v1/security/waf-profiles
POST /api/v1/security/waf-profiles
GET  /api/v1/security/events
```

---

## 8. Internal

```http
POST /internal/v1/security/keys/ensure
GET  /internal/v1/security/keys/{key_id}/version
```

Returns metadata + `wrapped_ref` **only** to internal token callers — still not raw key bytes.

---

## 9. Example flows

### 9.1 Create + rotate

1. `POST /keys` → version 1 ACTIVE  
2. `POST /keys/{id}/rotate` → version 2 ACTIVE, version 1 RETIRED  

### 9.2 Security matrix (expected)

| Case | Result |
|---|---|
| Unauthenticated | 401 |
| Authenticated, no permission | 403 |
| Wrong tenant | empty / not found |
| Correct tenant + `security.key.read` | 200 |
| Missing tenant on tenant key | 403 `SEC_TENANT_REQUIRED` |

---

## 10. Implementation checklist

- [ ] Envelope + error codes  
- [ ] Permissions  
- [ ] No secret on GET  
- [ ] RLS GUCs on HTTP  
- [ ] Idempotency  

---

## 11. Boundary reminders

p01 login, p03 secret values, p19 SIEM are out of this API.

---

## 12. Related documents

- Guide: [`SECURITY_GUIDE.md`](SECURITY_GUIDE.md)  
- Schema: [`SECURITY_SCHEMA.md`](SECURITY_SCHEMA.md)
