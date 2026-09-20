# JeslotERP Output Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — Alembic `f32a0b1c2d3e` / `f32b1c2d3e4f`; not Production  
**Package:** `platforms.p32_output`  
**PostgreSQL schema:** `output`  
**Depends on:** `p01_identity`, `p06_localization` (locale)  
**Integrates with:** `p15_notification` (delivery), `p09_document` (DMS store), `p08_file_media` (blobs)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`OUTPUT_SCHEMA.md`](OUTPUT_SCHEMA.md) · [`OUTPUT_API.md`](OUTPUT_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Output determination, templates, render, spool. Not invoice print. |

---

## 1. Purpose (enterprise)

`p32_output` is JeslotERP’s **output determination & rendering plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP NACE / SmartForms / spool**  
- **Dynamics / Salesforce document generation** (platform, not invoice)  

### Owns

| Domain | Examples |
|---|---|
| Templates | Key, locale, channel |
| Template versions | Immutable ACTIVE body |
| Determination | Pick template + channel from context |
| Render jobs | PDF / TEXT status + retry |
| Spool | Print/spool items |

### Does **not** own

| Concern | Owner |
|---|---|
| Blob bytes / virus scan | `p08_file_media` |
| DMS versioning / DIR | `p09_document` |
| Email/SMS delivery | `p15_notification` |
| Invoice / billing output | `b05_sales` / `b01_finance` (later) |
| Report datasets | `p24_reporting` |

### Critical split: Output vs Document vs Notification

| | **p32 Output** | **p09 Document** | **p15 Notification** |
|---|---|---|---|
| Question | What to render, which template/channel? | How is the file versioned in DMS? | How is it delivered? |
| Artifact | Determined job + spool | DIR / version | Message / inbox |

**Rule:** p32 determines/generates. p15 delivers. p09 stores documents. Do not implement invoice output.

---

## 2. Architectural position

```text
Caller context { output_type, locale, channel_hint, payload }
        │
        ▼
  Determination → template_version + channel
        │
        ▼
  Renderer port (TEXT builtin; WeasyPrint PDF fail-closed)
        │
        ├── p08 media_ref (UUID) — not invented
        └── p15 SendOrchestrator enqueue (EMAIL / IN_APP) — never SMTP in p32
        │
        ▼
  Spool + job status
```

---

## 3. Advanced design principles

1. **Determination first** — do not hardcode template keys in business code.  
2. **Locale from p06** — store locale code, not translated master.  
3. **ACTIVE template versions immutable.**  
4. **Retry** on render failure with max attempts.  
5. **Idempotent determine/render.**  
6. **PDF provider pending** is honest under pytest / empty `OUTPUT_PDF_ENGINE`. Never invent `%PDF` bytes.  
7. **No business forms in v1.**  
8. **PostgreSQL SoR.**

---

## 4. Core concepts

### 4.1 Job states

```text
PENDING → RENDERING → RENDERED → SPOOLED | FAILED
```

### 4.2 Channels

`PRINT` · `PDF` · `EMAIL` · `IN_APP` — EMAIL/IN_APP are *selection*; delivery is p15.

---

## 5. Security

### Permissions

| Code | Use |
|---|---|
| `output.template.read` | Templates |
| `output.template.manage` | Author |
| `output.determine` | Determination |
| `output.render` | Render |
| `output.spool.read` | Spool |
| `output.admin` | Admin |
| `output.*` | Wildcard |

### RLS

FORCE RLS on tenant jobs, spool, tenant template overlays.

---

## 6. Module layout

```text
platforms/p32_output/
  application/services/output_service.py
  application/ports/renderer.py
  infrastructure/http/… persistence/ renderers/text.py
```

---

## 7. Domain events

| Event | When |
|---|---|
| `output.job.rendered` / `failed` | Job |
| `output.template.activated` | Version |

Stream: `jesloterp:output:outbox`.

---

## 8. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs |
| P1 | Templates + versions + RLS |
| P2 | Determination |
| P3 | TEXT render + spool |
| P4 | PDF port stub |
| P5 | Tests + **SoR-Live** |

---

## 9. Definition of Done (enterprise)

- [x] Determination picks locale + channel  
- [x] TEXT renderer works  
- [x] PDF marked provider-pending, not faked as live  
- [x] P32-LIVE-001: named WeasyPrint adapter + p15 hand-off; no SMTP in p32  
- [x] Retry increments attempts  
- [x] No invoice objects  
- [x] Tenant RLS on jobs  
- [x] K28-33-001: list/get jobs DB-first on `AsyncSession` (empty is `[]` / 404)  

---

## 10. Anti-patterns

| Don’t | Do |
|---|---|
| SMTP send from p32 | Determine + hand to p15 |
| Store PDF bytes in `output` | media_ref → p08 |
| `invoice_print` tables | Generic `output_type` |

---

## 11. Related documents

- Schema: [`OUTPUT_SCHEMA.md`](OUTPUT_SCHEMA.md)  
- API: [`OUTPUT_API.md`](OUTPUT_API.md)  
- RTM: [`OUTPUT_RTM.md`](OUTPUT_RTM.md)  
- Implementation record: [`OUTPUT_IMPLEMENTATION_RECORD.md`](OUTPUT_IMPLEMENTATION_RECORD.md)  
- Notification: [`../15_notification/NOTIFICATION_GUIDE.md`](../15_notification/NOTIFICATION_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
