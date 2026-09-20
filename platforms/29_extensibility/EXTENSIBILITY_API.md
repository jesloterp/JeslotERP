# JeslotERP Extensibility Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live**  
**Package:** `platforms.p29_extensibility`  
**PostgreSQL schema:** `extensibility`  
**Public base:** `/api/v1/extensibility`  
**Internal base:** `/internal/v1/extensibility`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`EXTENSIBILITY_GUIDE.md`](EXTENSIBILITY_GUIDE.md) · [`EXTENSIBILITY_SCHEMA.md`](EXTENSIBILITY_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Points, handlers, bind, execute pipeline. |

---

## 1. Design principles (advanced)

1. **Execute is the product API.**  
2. **Allow-listed handlers only.**  
3. **Idempotent bind/execute.**  
4. **Fail-closed default.**

---

## 2. Common headers

Same as SECURITY_API §2.

---

## 3. Envelope

Standard success/error envelope.

---

## 4. Errors

```text
EXT_POINT_NOT_FOUND / EXT_HANDLER_UNKNOWN / EXT_HANDLER_DENIED
EXT_BINDING_NOT_FOUND / EXT_TIMEOUT / EXT_FAIL_CLOSED
EXT_FORBIDDEN
```

---

## 5. Permissions

See EXTENSIBILITY_GUIDE §5.

---

## 6. Catalog

```http
GET  /api/v1/extensibility/points
POST /api/v1/extensibility/points
GET  /api/v1/extensibility/handlers
GET  /api/v1/extensibility/bindings
POST /api/v1/extensibility/bindings
POST /api/v1/extensibility/bindings/{id}/activate
POST /api/v1/extensibility/bindings/{id}/deactivate
```

### 6.1 Create point

```json
{ "point_key": "platform.kernel.ping", "resource_type": "kernel", "description": "Health hook" }
```

### 6.2 Bind

```json
{ "point_key": "platform.kernel.ping", "handler_key": "ext.sample.noop", "phase": "AFTER", "priority": 10, "failure_policy": "FAIL_CLOSED" }
```

Unknown handler → `422 EXT_HANDLER_UNKNOWN`. Not allow-listed → `403 EXT_HANDLER_DENIED`.

---

## 7. Execute

```http
POST /api/v1/extensibility/execute
POST /internal/v1/extensibility/execute
```

```json
{
  "point_key": "platform.kernel.ping",
  "phase": "AFTER",
  "resource_type": "kernel",
  "resource_id": "00000000-0000-0000-0000-000000000001",
  "payload": { "ping": true }
}
```

**Response:**

```json
{ "ok": true, "results": [{ "handler_key": "ext.sample.noop", "status": "SUCCEEDED", "duration_ms": 1 }] }
```

Timeout → handler `TIMEOUT`; point policy may `EXT_FAIL_CLOSED` (409).

---

## 8. Executions audit

```http
GET /api/v1/extensibility/executions
```

Permission: `extensibility.audit.read`.

---

## 9. Security matrix

| Case | Result |
|---|---|
| Unauthenticated execute | 401 |
| No `extensibility.execute` | 403 |
| Unknown handler bind | 422 |
| Inactive binding | skipped, not run |

---

## 10. Implementation checklist

- [ ] No eval/exec  
- [ ] Allow-list enforced  
- [ ] Priority order  
- [ ] RLS on executions  

---

## 11. Boundary reminders

No CRM/sales handlers. p11 remains the decision engine.

---

## 12. Related documents

- Guide: [`EXTENSIBILITY_GUIDE.md`](EXTENSIBILITY_GUIDE.md)  
- Schema: [`EXTENSIBILITY_SCHEMA.md`](EXTENSIBILITY_SCHEMA.md)
