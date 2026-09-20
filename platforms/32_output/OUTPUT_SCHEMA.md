# JeslotERP Output Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — lean SoR (AUD-022); Alembic `f32a0b1c2d3e` / `f32b1c2d3e4f`  
**Package:** `platforms.p32_output`  
**PostgreSQL schema:** `output`  
**Companion:** [`OUTPUT_GUIDE.md`](OUTPUT_GUIDE.md) · [`OUTPUT_API.md`](OUTPUT_API.md)

> Runtime models: `platforms/p32_output/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `output` (never `p32`) |
| Tables | `out_*` |
| Cross-schema | `media_ref` / `document_ref` UUIDs — no FK |
| RLS | FORCE on tenant jobs/spool/overlays |

---

## 2. Complete table inventory (**6 domain + 2 plumbing**)

| # | Table | Purpose |
|---|---|---|
| 1 | `out_template` | Template catalog |
| 2 | `out_template_version` | Immutable versions |
| 3 | `out_determination` | Rules: type+locale+channel |
| 4 | `out_job` | Render jobs |
| 5 | `out_spool` | Spool items |
| 6 | `out_channel_binding` | Channel → p15 ref |
| 7 | `out_outbox` | Outbox |
| 8 | `out_idempotency_key` | Idempotency |

---

## 3. Enumerations

| Enum | Values |
|---|---|
| Channel | `PRINT`, `PDF`, `EMAIL`, `IN_APP` |
| Job status | `PENDING`, `RENDERING`, `RENDERED`, `SPOOLED`, `FAILED` |
| Renderer | `TEXT`, `PDF` |

---

## 4. Detailed tables

### 4.1 `out_template`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | |
| `template_key` | VARCHAR(120) | |
| `output_type` | VARCHAR(80) | Generic — not `invoice` |
| `status` | VARCHAR(20) | |

### 4.2 `out_template_version`

| Column | Type | Notes |
|---|---|---|
| `template_id` | UUID | |
| `version_no` | INTEGER | |
| `locale` | VARCHAR(20) | |
| `body` | TEXT | TEXT renderer source |
| `is_active` | BOOLEAN | |

### 4.3 `out_determination`

| Column | Type | Notes |
|---|---|---|
| `output_type` | VARCHAR(80) | |
| `locale` | VARCHAR(20) | |
| `channel` | VARCHAR(20) | |
| `template_id` | UUID | |
| `priority` | INTEGER | |

### 4.4 `out_job`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `template_version_id` | UUID | |
| `channel` | VARCHAR(20) | |
| `status` | VARCHAR(20) | |
| `attempts` | INTEGER | |
| `max_attempts` | INTEGER | Default 3 |
| `media_ref` | UUID NULL | p08 |
| `error_code` | VARCHAR(80) NULL | |

### 4.5 `out_spool`

| Column | Type | Notes |
|---|---|---|
| `job_id` | UUID | |
| `channel` | VARCHAR(20) | |
| `status` | VARCHAR(20) | |

### 4.6 `out_channel_binding`

| Column | Type | Notes |
|---|---|---|
| `channel` | VARCHAR(20) | |
| `notify_ref` | UUID NULL | p15 template/channel id |

---

## 5. RLS summary

Jobs, spool, tenant templates: FORCE RLS.

---

## 6. Seed minimum

Template `platform.notice` + TEXT version `en`. Permissions `output.*`.

---

## 7. ER overview

```text
out_template 1──* out_template_version
      │
      └──* out_determination
out_job 1──* out_spool
```

---

## 8. Implementation notes

Alembic `f32a0b1c2d3e` / `f32b1c2d3e4f`. PDF renderer is `WeasyPrintPdfAdapter` (pytest pending).
