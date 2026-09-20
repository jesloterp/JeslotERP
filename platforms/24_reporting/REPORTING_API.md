# JeslotERP Reporting Platform — Complete API Endpoints

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — list/create datasets and reports are Postgres-first; empty list is `[]`. DWH is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p24_reporting`  
**Base path:** `/api/v1/reporting`  
**Companion:** [`REPORTING_GUIDE.md`](REPORTING_GUIDE.md) · [`REPORTING_SCHEMA.md`](REPORTING_SCHEMA.md)

---

## 0. Conventions

### Headers

| Header | Required | Notes |
|---|---|---|
| `Authorization` | Yes | Bearer JWT |
| `X-Tenant-Id` | Yes | Tenant scope |
| `X-Branch-Id` | Optional | RLS claim input |
| `X-Correlation-Id` | Recommended | Trace |
| `Idempotency-Key` | Mutating / schedule fire | Required for create run/export when async |

### Envelope

`StandardResponse` with `data` / `error` / `meta`.

### Common errors

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `INVALID_PARAMETERS` | Param validation failed |
| 403 | `FORBIDDEN` / `RLS_DENIED` | Share or row policy |
| 404 | `NOT_FOUND` | Unknown report/run |
| 409 | `CONFLICT` | Duplicate key / sealed snapshot |
| 413 | `BUDGET_EXCEEDED` | Rows/time/bytes |
| 422 | `VALIDATION_ERROR` | Def compile errors |
| 423 | `DRAFT_NOT_RUNNABLE` | Unpublished in prod |
| 429 | `RATE_LIMITED` | Run flood |
| 409 | `RUN_CANCELLED` | Cancelled |

---

## 1. Catalog & folders

### 1.1 Folders

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/folders` | `reporting.catalog.read` |
| `POST` | `/api/v1/reporting/folders` | `reporting.share.manage` |
| `PATCH` | `/api/v1/reporting/folders/{folder_id}` | `reporting.share.manage` |
| `DELETE` | `/api/v1/reporting/folders/{folder_id}` | `reporting.share.manage` |

**GET tree query:** `parent_id`, `depth`

### 1.2 Catalog browse

`GET /api/v1/reporting/catalog`

**Permission:** `reporting.catalog.read`  
**Query:** `q`, `folder_id`, `tag`, `lifecycle`, `layout_kind`, `page`, `page_size`

**Item:** `id`, `kind` (`REPORT`|`FOLDER`), `name`, `report_key`, `lifecycle`, `owner`, `updated_at`, `is_favorite`

### 1.3 Favorites

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/favorites` | `reporting.catalog.read` |
| `PUT` | `/api/v1/reporting/favorites/{report_id}` | `reporting.catalog.read` |
| `DELETE` | `/api/v1/reporting/favorites/{report_id}` | `reporting.catalog.read` |

---

## 2. Datasets

### 2.1 CRUD & versions

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/datasets` | `reporting.dataset.read` |
| `POST` | `/api/v1/reporting/datasets` | `reporting.dataset.manage` |
| `GET` | `/api/v1/reporting/datasets/{dataset_id}` | `reporting.dataset.read` |
| `PATCH` | `/api/v1/reporting/datasets/{dataset_id}` | `reporting.dataset.manage` |
| `POST` | `/api/v1/reporting/datasets/{dataset_id}/versions` | `reporting.dataset.manage` |
| `POST` | `/api/v1/reporting/datasets/{dataset_id}/versions/{version}/publish` | `reporting.dataset.manage` |
| `GET` | `/api/v1/reporting/datasets/{dataset_id}/fields` | `reporting.dataset.read` |
| `PUT` | `/api/v1/reporting/datasets/{dataset_id}/rls` | `reporting.dataset.manage` |
| `PUT` | `/api/v1/reporting/datasets/{dataset_id}/budget` | `reporting.admin` |

**POST dataset (sketch):**

```json
{
  "dataset_key": "freight.invoices",
  "name": "Freight invoices",
  "primary_entity_key": "freight_invoice",
  "sources": [{ "alias": "inv", "entity_key": "freight_invoice" }],
  "fields": [
    { "field_key": "invoice_no", "role": "DIMENSION", "source_path": "inv.number", "data_type": "STRING" },
    { "field_key": "amount", "role": "MEASURE", "source_path": "inv.amount", "data_type": "DECIMAL" }
  ],
  "measures": [
    { "measure_key": "amount_sum", "agg": "SUM", "expression": "SUM(amount)" }
  ]
}
```

**Publish response:** `version`, `checksum`.

### 2.2 Preview (admin/designer)

`POST /api/v1/reporting/datasets/{dataset_id}/preview`  
**Permission:** `reporting.dataset.manage`  
**Body:** `{ "parameters": {}, "limit": 50 }`  
Respects RLS; hard-capped by `sync_max_rows`.

---

## 3. Report definitions

### 3.1 Reports

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/reports` | `reporting.catalog.read` |
| `POST` | `/api/v1/reporting/reports` | `reporting.def.manage` |
| `GET` | `/api/v1/reporting/reports/{report_id}` | `reporting.catalog.read` |
| `PATCH` | `/api/v1/reporting/reports/{report_id}` | `reporting.def.manage` |
| `POST` | `/api/v1/reporting/reports/{report_id}/versions` | `reporting.def.manage` |
| `POST` | `/api/v1/reporting/reports/{report_id}/publish` | `reporting.def.manage` |
| `POST` | `/api/v1/reporting/reports/{report_id}/retire` | `reporting.def.manage` |
| `POST` | `/api/v1/reporting/reports/{report_id}/compile` | `reporting.def.manage` |

**POST create:**

```json
{
  "report_key": "gst.outward.register",
  "name": "GST Outward Register",
  "folder_id": "…",
  "layout_kind": "TABLE",
  "dataset_id": "…",
  "columns": [
    { "field_key": "invoice_no", "ordinal": 1 },
    { "measure_key": "amount_sum", "ordinal": 2 }
  ],
  "parameters": [
    { "param_key": "from_date", "param_type": "DATE", "is_required": true },
    { "param_key": "to_date", "param_type": "DATE", "is_required": true }
  ]
}
```

**Compile:** returns validation errors or `plan_checksum` without executing.

### 3.2 Move / tags

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/reporting/reports/{report_id}/move` | `reporting.share.manage` |
| `PUT` | `/api/v1/reporting/reports/{report_id}/tags` | `reporting.def.manage` |

---

## 4. Parameters & variants

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/reports/{report_id}/parameters` | `reporting.catalog.read` |
| `GET` | `/api/v1/reporting/reports/{report_id}/variants` | `reporting.catalog.read` |
| `POST` | `/api/v1/reporting/reports/{report_id}/variants` | `reporting.run` |
| `PATCH` | `/api/v1/reporting/variants/{variant_id}` | `reporting.run` |
| `DELETE` | `/api/v1/reporting/variants/{variant_id}` | `reporting.run` |

**Variant body:**

```json
{
  "name": "This month – HO branch",
  "values": { "from_date": "2026-09-01", "to_date": "2026-09-30", "branch_id": "…" },
  "columns": { "hidden": ["internal_code"] },
  "is_shared": false
}
```

---

## 5. Execution (runs)

### 5.1 Start run

`POST /api/v1/reporting/reports/{report_id}/runs`  
**Permission:** `reporting.run`  
**Header:** `Idempotency-Key` recommended  

```json
{
  "variant_id": null,
  "parameters": { "from_date": "2026-09-01", "to_date": "2026-09-09" },
  "mode": "AUTO"
}
```

`mode`: `SYNC` | `ASYNC` | `AUTO` (AUTO chooses by budget estimate).

**Sync response (`202`/`200`):** run summary + first page of rows (capped).  
**Async response (`202`):** `{ "run_id", "status": "QUEUED", "job_id" }`.

### 5.2 Run lifecycle

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/runs/{run_id}` | `reporting.run` |
| `GET` | `/api/v1/reporting/runs/{run_id}/pages/{page}` | `reporting.run` |
| `GET` | `/api/v1/reporting/runs/{run_id}/metrics` | `reporting.run` |
| `POST` | `/api/v1/reporting/runs/{run_id}/cancel` | `reporting.run` |
| `GET` | `/api/v1/reporting/runs` | `reporting.run` |

**List query:** `report_id`, `status`, `from`, `to`, `page`

**Page response:** `{ "columns": […], "rows": […], "page", "page_size", "total_rows" }` — never unbounded.

### 5.3 Internal worker complete

`POST /api/v1/reporting/internal/runs/{run_id}/complete`  
**Auth:** service  
Body: status, row_count, metrics, error.

---

## 6. Exports & downloads

### 6.1 Create export

`POST /api/v1/reporting/runs/{run_id}/exports`  
**Permission:** `reporting.export`  

```json
{
  "format": "XLSX",
  "options": { "locale": "en-IN", "include_totals": true }
}
```

**Response:** `{ "export_id", "status": "QUEUED" }`

### 6.2 Export status & artifact

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/exports/{export_id}` | `reporting.export` |
| `GET` | `/api/v1/reporting/exports/{export_id}/download` | `reporting.export` |
| `POST` | `/api/v1/reporting/exports/{export_id}/deliver` | `reporting.export` |

**Download:** returns redirect/signed URL to p08; issues `download_grant`.  
**Deliver:** `{ "channel": "EMAIL", "recipient_user_ids": ["…"] }` → p15.

---

## 7. Subscriptions & schedules

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/subscriptions` | `reporting.schedule.manage` |
| `POST` | `/api/v1/reporting/subscriptions` | `reporting.schedule.manage` |
| `PATCH` | `/api/v1/reporting/subscriptions/{id}` | `reporting.schedule.manage` |
| `POST` | `/api/v1/reporting/subscriptions/{id}/pause` | `reporting.schedule.manage` |
| `POST` | `/api/v1/reporting/subscriptions/{id}/resume` | `reporting.schedule.manage` |
| `GET` | `/api/v1/reporting/subscriptions/{id}/history` | `reporting.schedule.manage` |
| `POST` | `/api/v1/reporting/internal/subscriptions/{id}/fire` | internal (p17) |

**POST create:**

```json
{
  "report_id": "…",
  "variant_id": "…",
  "format": "PDF",
  "timezone": "Asia/Kolkata",
  "scheduler_cron": "0 7 * * 1",
  "recipients": [{ "kind": "USER", "id": "…" }, { "kind": "EMAIL", "address": "finance@example.com" }],
  "burst": { "slice_field": "branch_id" }
}
```

Platform registers/updates p17 schedule; fire endpoint is idempotent per `idempotency_window_key`.

---

## 8. Sharing

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/reports/{report_id}/shares` | `reporting.share.manage` |
| `PUT` | `/api/v1/reporting/reports/{report_id}/shares` | `reporting.share.manage` |
| `POST` | `/api/v1/reporting/reports/{report_id}/share-links` | `reporting.share.manage` |
| `DELETE` | `/api/v1/reporting/share-links/{link_id}` | `reporting.share.manage` |

**PUT shares:**

```json
{
  "shares": [
    { "scope": "ROLE", "principal_id": "…", "can_run": true, "can_export": true, "can_edit": false },
    { "scope": "USER", "principal_id": "…", "can_run": true, "can_export": false, "can_edit": false }
  ]
}
```

**Share link:** time-bound token; run/export still enforce dataset RLS.

---

## 9. Snapshots

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/reporting/reports/{report_id}/snapshots` | `reporting.snapshot.manage` |
| `GET` | `/api/v1/reporting/snapshots` | `reporting.catalog.read` |
| `GET` | `/api/v1/reporting/snapshots/{snapshot_id}` | `reporting.catalog.read` |
| `POST` | `/api/v1/reporting/snapshots/{snapshot_id}/seal` | `reporting.snapshot.manage` |
| `GET` | `/api/v1/reporting/snapshots/{snapshot_id}/download` | `reporting.export` |

**Create:**

```json
{
  "period_key": "2026-08",
  "parameters": { "from_date": "2026-08-01", "to_date": "2026-08-31" },
  "format": "XLSX"
}
```

After `seal`, mutations return `409 CONFLICT`.

---

## 10. Packs, budgets, admin

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/reporting/packages` | `reporting.admin` |
| `POST` | `/api/v1/reporting/packages/{package_key}/apply` | `reporting.admin` |
| `GET` | `/api/v1/reporting/admin/budgets` | `reporting.admin` |
| `POST` | `/api/v1/reporting/admin/purge-runs` | `reporting.admin` |
| `GET` | `/api/v1/reporting/report-types` | `reporting.catalog.read` |

**Purge body:** `{ "older_than": "2026-01-01", "statuses": ["SUCCEEDED", "FAILED"] }` — does not purge sealed snapshots.

---

## 11. Dashboard / search contracts

### 11.1 Dataset binding for p25

`GET /api/v1/reporting/datasets/{dataset_id}/binding-contract`  
**Permission:** `reporting.dataset.read`  

Returns stable field/measure keys, param schema, RLS notes for widget authors.

### 11.2 Catalog index push (internal)

`POST /api/v1/reporting/internal/catalog/reindex`  
Pushes report docs to p18 (id, name, folder, tags, tenant).

---

## 12. Permission matrix (summary)

| Surface | Min permission |
|---|---|
| Browse / favorites | `reporting.catalog.read` |
| Datasets read | `reporting.dataset.read` |
| Datasets write | `reporting.dataset.manage` |
| Defs write | `reporting.def.manage` |
| Run / variants | `reporting.run` |
| Export / download | `reporting.export` |
| Schedules | `reporting.schedule.manage` |
| Shares | `reporting.share.manage` |
| Snapshots | `reporting.snapshot.manage` |
| Packs / purge | `reporting.admin` |

---

## 13. Example flows

### 13.1 Interactive run

1. `GET /catalog?q=gst`  
2. `GET /reports/{id}/parameters`  
3. `POST /reports/{id}/runs` `mode=SYNC`  
4. Render page 1; `GET .../pages/2` as needed  
5. Optional `POST .../exports` → download

### 13.2 Monday PDF subscription

1. Publish report + variant  
2. `POST /subscriptions` with cron + recipients  
3. p17 fires → `internal/.../fire` → async run → export → p15 notify  
4. User opens download grant

### 13.3 Month-end seal

1. `POST /reports/{id}/snapshots` for `period_key`  
2. Review run/artifact  
3. `POST /snapshots/{id}/seal`  
4. Further edits blocked; auditors download sealed artifact

---

## 14. Related documents

- Guide: [`REPORTING_GUIDE.md`](REPORTING_GUIDE.md)  
- Schema: [`REPORTING_SCHEMA.md`](REPORTING_SCHEMA.md)  
- Metadata: [`../05_metadata/METADATA_API.md`](../05_metadata/METADATA_API.md)  
- File media: [`../08_file_media/FILE_MEDIA_API.md`](../08_file_media/FILE_MEDIA_API.md)  
- Scheduler: [`../17_scheduler/SCHEDULER_API.md`](../17_scheduler/SCHEDULER_API.md)  
- Dashboard: [`../25_dashboard/DASHBOARD_GUIDE.md`](../25_dashboard/DASHBOARD_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
