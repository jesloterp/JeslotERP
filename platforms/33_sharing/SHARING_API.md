# JeslotERP Sharing Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live**  
**Package:** `platforms.p33_sharing`  
**PostgreSQL schema:** `sharing`  
**Public base:** `/api/v1/sharing`  
**Internal base:** `/internal/v1/sharing`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`SHARING_GUIDE.md`](SHARING_GUIDE.md) · [`SHARING_SCHEMA.md`](SHARING_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Teams, grants, evaluate, hierarchy, rules. |
| **1.1** | **2026-09-12** | P33-LIVE-001: evaluate is DB-first on `AsyncSession`. |

---

## 1. Design principles (advanced)

1. **Evaluate is the product API.**  
2. **Generic resource identifiers.**  
3. **Idempotent grant.**  
4. **Fail-closed.**

---

## 2. Common headers

Standard + `Idempotency-Key`.

---

## 3. Envelope

Standard.

---

## 4. Errors

```text
SHR_TEAM_NOT_FOUND / SHR_GRANT_NOT_FOUND / SHR_RULE_NOT_FOUND
SHR_FORBIDDEN / SHR_TENANT_REQUIRED
```

---

## 5. Permissions

See SHARING_GUIDE §5.

---

## 6. Teams / hierarchy / rules

```http
GET  /api/v1/sharing/teams
POST /api/v1/sharing/teams
POST /api/v1/sharing/teams/{id}/members
DELETE /api/v1/sharing/teams/{id}/members/{user_ref}
GET  /api/v1/sharing/hierarchy
POST /api/v1/sharing/hierarchy
GET  /api/v1/sharing/rules
POST /api/v1/sharing/rules
POST /api/v1/sharing/rules/{id}/activate
```

---

## 7. Grants

```http
GET  /api/v1/sharing/grants
POST /api/v1/sharing/grants
POST /api/v1/sharing/grants/{id}/revoke
```

```json
{
  "resource_type": "kernel.record",
  "resource_id": "…",
  "grantee_kind": "USER",
  "grantee_id": "…",
  "access": "READ",
  "reason": "MANUAL"
}
```

Duplicate grant → 200 same `grant_id` (idempotent).

---

## 8. Evaluate

```http
POST /api/v1/sharing/evaluate
POST /internal/v1/sharing/evaluate
```

```json
{
  "user_ref": "…",
  "resource_type": "kernel.record",
  "resource_id": "…",
  "access": "READ"
}
```

**Response:**

```json
{ "allowed": true, "reason": "OWNER", "grant_ids": ["…"] }
```

Reasons checked in order: OWNER, MANUAL, TEAM, RULE, HIERARCHY. None → `{ "allowed": false, "reason": null }`.

HTTP evaluate is DB-first: when the request session is `AsyncSession`, grants come from `fetch_grants`. An empty catalog denies (no process-ledger fallback). TestClient/`AsyncMock` still uses the in-memory ledger.

---

## 9. Security matrix

| Case | Result |
|---|---|
| Unauthenticated | 401 |
| No `sharing.evaluate` | 403 |
| Wrong tenant grants | empty / deny |
| Stranger evaluate | `allowed: false` |

---

## 10. Implementation checklist

- [ ] Owner / manual / team / hierarchy / rule  
- [ ] Unique grant  
- [ ] RLS  
- [ ] No sales-order types  

---

## 11. Boundary reminders

p01 = tenant/org access. p33 = this record.

---

## 12. Related documents

- Guide: [`SHARING_GUIDE.md`](SHARING_GUIDE.md)  
- Schema: [`SHARING_SCHEMA.md`](SHARING_SCHEMA.md)
