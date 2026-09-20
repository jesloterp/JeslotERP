# JeslotERP Output Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live**  
**Package:** `platforms.p32_output`  
**PostgreSQL schema:** `output`  
**Public base:** `/api/v1/output`  
**Internal base:** `/internal/v1/output`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`OUTPUT_GUIDE.md`](OUTPUT_GUIDE.md) · [`OUTPUT_SCHEMA.md`](OUTPUT_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Templates, determine, render, spool. |

---

## 1. Design principles (advanced)

1. **Determine then render.**  
2. **TEXT builtin; PDF is a named WeasyPrint port (fail-closed).**  
3. **Idempotent determine/render.**  
4. **No invoice resources.**

---

## 2. Common headers

Standard + `Idempotency-Key`.

---

## 3. Envelope

Standard.

---

## 4. Errors

```text
OUT_TEMPLATE_NOT_FOUND / OUT_NO_DETERMINATION / OUT_RENDER_FAILED
OUT_PDF_PROVIDER_PENDING / OUT_FORBIDDEN
```

---

## 5. Permissions

See OUTPUT_GUIDE §5.

---

## 6. Templates

```http
GET  /api/v1/output/templates
POST /api/v1/output/templates
POST /api/v1/output/templates/{id}/versions
POST /api/v1/output/templates/{id}/versions/{n}/activate
GET  /api/v1/output/determinations
POST /api/v1/output/determinations
```

```json
{ "template_key": "platform.notice", "output_type": "NOTICE" }
```

Version body: `{ "locale": "en", "body": "Hello {{name}}" }`.

---

## 7. Determine / render / spool

```http
POST /api/v1/output/determine
POST /api/v1/output/render
GET  /api/v1/output/jobs
GET  /api/v1/output/jobs/{id}
POST /api/v1/output/jobs/{id}/retry
GET  /api/v1/output/spool
```

### 7.1 Determine

```json
{ "output_type": "NOTICE", "locale": "en", "channel": "PRINT" }
```

**Response:** `{ "template_id", "template_version_id", "channel" }`.  
No match → `422 OUT_NO_DETERMINATION`.

### 7.2 Render

```json
{ "template_version_id": "…", "channel": "PRINT", "payload": { "name": "A" }, "renderer": "TEXT" }
```

TEXT substitutes `{{name}}`.  
`renderer=PDF` → `422 OUT_PDF_PROVIDER_PENDING` under pytest / empty `OUTPUT_PDF_ENGINE` (never invents `%PDF` bytes).  
`POST /api/v1/output/renderers/pdf/test-connection` → `PROVIDER_PENDING` in that case.

EMAIL / IN_APP render calls p15 `SendOrchestrator` and sets `notify_ref`. PRINT does not. p32 never SMTP.

---

## 8. Internal

```http
POST /internal/v1/output/determine
POST /internal/v1/output/render
```

---

## 9. Security matrix

| Case | Result |
|---|---|
| Unauthenticated | 401 |
| Wrong tenant jobs | empty |
| PDF without provider | 422 pending |

---

## 10. Implementation checklist

- [x] Determination  
- [x] TEXT render  
- [x] Retry attempts  
- [x] No invoice types  
- [x] PDF port + p15 hand-off (no SMTP)

---

## 11. Boundary reminders

p15 delivers; p08/p09 store files. p32 does not SMTP.

---

## 12. Related documents

- Guide: [`OUTPUT_GUIDE.md`](OUTPUT_GUIDE.md)  
- Schema: [`OUTPUT_SCHEMA.md`](OUTPUT_SCHEMA.md)
