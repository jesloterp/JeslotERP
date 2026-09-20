# JeslotERP Dashboard Platform — Complete API Endpoints

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — folders/boards/widgets list+create are Postgres-first; empty list is `[]`. Not Production.  
**Package:** `platforms.p25_dashboard`  
**Base path:** `/api/v1/dashboard`  
**Companion:** [`DASHBOARD_GUIDE.md`](DASHBOARD_GUIDE.md) · [`DASHBOARD_SCHEMA.md`](DASHBOARD_SCHEMA.md)

---

## 0. Conventions

### Headers

| Header | Required | Notes |
|---|---|---|
| `Authorization` | Yes | Bearer JWT |
| `X-Tenant-Id` | Yes | Tenant scope |
| `X-Branch-Id` | Optional | Filter/RLS claim |
| `X-Correlation-Id` | Recommended | Trace |
| `Idempotency-Key` | Publish / clone | Recommended |

### Envelope

`StandardResponse` with `data` / `error` / `meta`.

### Common errors

| HTTP | Code | Meaning |
|---|---|---|
| 403 | `FORBIDDEN` / `RLS_DENIED` | Share or dataset RLS |
| 404 | `NOT_FOUND` | Unknown board/widget |
| 409 | `CONFLICT` | Duplicate board_key / published immutable |
| 422 | `VALIDATION_ERROR` / `BINDING_INVALID` | Bad layout/binding |
| 423 | `DRAFT_NOT_VIEWABLE` | Policy blocks draft for non-editors |
| 424 | `FEATURE_DISABLED` | Widget type gated off |
| 413 | `BUDGET_EXCEEDED` | Widget query budget |
| 429 | `RATE_LIMITED` | Refresh flood |

---

## 1. Catalog & folders

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/folders` | `dashboard.catalog.read` |
| `POST` | `/api/v1/dashboard/folders` | `dashboard.board.manage` |
| `GET` | `/api/v1/dashboard/catalog` | `dashboard.catalog.read` |
| `GET` | `/api/v1/dashboard/favorites` | `dashboard.catalog.read` |
| `PUT` | `/api/v1/dashboard/favorites/{board_id}` | `dashboard.catalog.read` |
| `DELETE` | `/api/v1/dashboard/favorites/{board_id}` | `dashboard.catalog.read` |

**Catalog query:** `q`, `folder_id`, `scope`, `tag`, `lifecycle`, `page`, `page_size`

**Item:** `id`, `board_key`, `name`, `scope`, `lifecycle`, `is_favorite`, `updated_at`

---

## 2. Boards

### 2.1 CRUD & versions

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/boards` | `dashboard.catalog.read` |
| `POST` | `/api/v1/dashboard/boards` | `dashboard.board.manage` |
| `GET` | `/api/v1/dashboard/boards/{board_id}` | `dashboard.catalog.read` |
| `PATCH` | `/api/v1/dashboard/boards/{board_id}` | `dashboard.board.manage` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/versions` | `dashboard.board.manage` |
| `GET` | `/api/v1/dashboard/boards/{board_id}/versions` | `dashboard.catalog.read` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/publish` | `dashboard.publish` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/retire` | `dashboard.board.manage` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/clone` | `dashboard.board.manage` |

**POST create:**

```json
{
  "board_key": "ops.dispatch.home",
  "name": "Dispatch Home",
  "folder_id": "…",
  "scope": "TENANT",
  "theme": { "density": "compact" },
  "default_filters": [
    { "filter_key": "date_range", "param_type": "DATE", "default_json": { "preset": "TODAY" } },
    { "filter_key": "branch_id", "param_type": "BRANCH", "is_global": true }
  ]
}
```

**GET board** returns published version for viewers; editors may `?version=draft`.

**Clone body:** `{ "board_key": "ops.dispatch.home.custom", "name": "…" }` — records lineage.

### 2.2 Runtime board payload (shell)

`GET /api/v1/dashboard/boards/{board_id}/runtime`  
**Permission:** `dashboard.catalog.read`  

Returns layout sections, widget placements, filter defs, refresh policies — **without** heavy data series (use widget data APIs).

---

## 3. Layout & widgets

### 3.1 Layout

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/boards/{board_id}/layout` | `dashboard.catalog.read` |
| `PUT` | `/api/v1/dashboard/boards/{board_id}/layout` | `dashboard.widget.manage` |

**PUT layout** applies to **draft** version:

```json
{
  "sections": [
    {
      "section_key": "today",
      "title_key": "dashboard.section.today",
      "ordinal": 1,
      "columns": 12
    }
  ],
  "breakpoints": {
    "lg": { "columns": 12, "row_height": 32 }
  }
}
```

### 3.2 Widgets

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/widget-types` | `dashboard.catalog.read` |
| `GET` | `/api/v1/dashboard/boards/{board_id}/widgets` | `dashboard.catalog.read` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/widgets` | `dashboard.widget.manage` |
| `PATCH` | `/api/v1/dashboard/widgets/{widget_id}` | `dashboard.widget.manage` |
| `DELETE` | `/api/v1/dashboard/widgets/{widget_id}` | `dashboard.widget.manage` |
| `PUT` | `/api/v1/dashboard/widgets/{widget_id}/placement` | `dashboard.widget.manage` |
| `PUT` | `/api/v1/dashboard/widgets/{widget_id}/binding` | `dashboard.widget.manage` |

**POST widget:**

```json
{
  "widget_key": "kpi.open_trips",
  "type_key": "kpi.v1",
  "section_key": "today",
  "title_key": "dashboard.kpi.open_trips",
  "placement": { "lg": { "x": 0, "y": 0, "w": 3, "h": 2 } },
  "config": { "emphasis": "high" },
  "binding": {
    "binding_mode": "DATASET",
    "dataset_id": "…",
    "measures": [{ "measure_key": "trip_count", "format": "number" }],
    "param_map": { "branch_id": "{{filters.branch_id}}" }
  },
  "refresh_policy_key": "INTERVAL_60"
}
```

**Binding validation** calls p24 binding-contract; invalid measure → `422 BINDING_INVALID`.

---

## 4. Filters

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/boards/{board_id}/filters` | `dashboard.catalog.read` |
| `PUT` | `/api/v1/dashboard/boards/{board_id}/filters` | `dashboard.widget.manage` |
| `GET` | `/api/v1/dashboard/boards/{board_id}/filter-presets` | `dashboard.personalize` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/filter-presets` | `dashboard.personalize` |
| `DELETE` | `/api/v1/dashboard/filter-presets/{preset_id}` | `dashboard.personalize` |

---

## 5. Widget data & refresh

### 5.1 Query widget data

`POST /api/v1/dashboard/widgets/{widget_id}/data`  
**Permission:** `dashboard.catalog.read`  

```json
{
  "filters": {
    "date_range": { "from": "2026-09-09", "to": "2026-09-09" },
    "branch_id": "…"
  },
  "force_refresh": false
}
```

**Response:**

```json
{
  "widget_id": "…",
  "generated_at": "…",
  "cache": { "hit": true, "age_seconds": 12, "stale": false },
  "columns": ["trip_count"],
  "rows": [{ "trip_count": 42 }],
  "threshold": { "severity": null }
}
```

Server merges board global filters + widget filters → p24 preview/run (budgeted) as **running user**.

### 5.2 Batch data (board open)

`POST /api/v1/dashboard/boards/{board_id}/data`  
**Permission:** `dashboard.catalog.read`  

```json
{
  "filters": { "branch_id": "…" },
  "widget_ids": null
}
```

Returns map of widget_id → data payloads; parallelized with bulkhead; respects per-widget budgets.

### 5.3 Refresh

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/dashboard/widgets/{widget_id}/refresh` | `dashboard.catalog.read` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/refresh` | `dashboard.catalog.read` |
| `POST` | `/api/v1/dashboard/internal/refresh/{board_id}` | internal (p17/p14) |

Invalidate cache tags and optionally enqueue warm job.

---

## 6. Personalization

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/boards/{board_id}/personalization` | `dashboard.personalize` |
| `PUT` | `/api/v1/dashboard/boards/{board_id}/personalization` | `dashboard.personalize` |
| `DELETE` | `/api/v1/dashboard/boards/{board_id}/personalization` | `dashboard.personalize` |

**PUT body:**

```json
{
  "hidden_widget_keys": ["markdown.help"],
  "placements": {
    "kpi.open_trips": { "lg": { "x": 3, "y": 0, "w": 3, "h": 2 } }
  },
  "filter_preset_id": "…"
}
```

Rejected if widget `is_locked` or board disallows reorder → `403`.

---

## 7. Home assignment & preferences

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/home` | `dashboard.catalog.read` |
| `GET` | `/api/v1/dashboard/home-assignments` | `dashboard.share.manage` |
| `PUT` | `/api/v1/dashboard/home-assignments` | `dashboard.share.manage` |
| `GET` | `/api/v1/dashboard/preferences` | `dashboard.personalize` |
| `PUT` | `/api/v1/dashboard/preferences` | `dashboard.personalize` |
| `GET` | `/api/v1/dashboard/recents` | `dashboard.catalog.read` |
| `PUT` | `/api/v1/dashboard/pins/{board_id}` | `dashboard.catalog.read` |

**GET /home** resolves USER → ROLE → tenant default → system sample.

**PUT home-assignments:**

```json
{
  "assignments": [
    { "principal_kind": "ROLE", "principal_id": "…", "board_id": "…", "priority": 100 },
    { "principal_kind": "USER", "principal_id": "…", "board_id": "…", "priority": 200 }
  ]
}
```

---

## 8. Sharing & links

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/boards/{board_id}/shares` | `dashboard.share.manage` |
| `PUT` | `/api/v1/dashboard/boards/{board_id}/shares` | `dashboard.share.manage` |
| `POST` | `/api/v1/dashboard/boards/{board_id}/share-links` | `dashboard.share.manage` |
| `DELETE` | `/api/v1/dashboard/share-links/{link_id}` | `dashboard.share.manage` |

**PUT shares:** scopes ROLE/USER with `can_view` / `can_edit` / `can_share`.  
Link shares still enforce dataset RLS on widget data.

---

## 9. Drill-through

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/widgets/{widget_id}/drill-targets` | `dashboard.catalog.read` |
| `PUT` | `/api/v1/dashboard/widgets/{widget_id}/drill-targets` | `dashboard.widget.manage` |
| `POST` | `/api/v1/dashboard/widgets/{widget_id}/drill` | `dashboard.catalog.read` |

**POST drill:**

```json
{
  "target_key": "open_report",
  "context": { "branch_id": "…", "status": "OPEN" }
}
```

**Response:** `{ "kind": "REPORT", "report_id": "…", "parameters": {…} }` or `{ "kind": "ENTITY_ROUTE", "path": "/trips/…" }`  
Frontend navigates; report execution stays in p24.

---

## 10. Threshold alerts

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/widgets/{widget_id}/thresholds` | `dashboard.alert.manage` |
| `PUT` | `/api/v1/dashboard/widgets/{widget_id}/thresholds` | `dashboard.alert.manage` |
| `GET` | `/api/v1/dashboard/alerts` | `dashboard.alert.manage` |
| `POST` | `/api/v1/dashboard/alerts/{alert_event_id}/ack` | `dashboard.alert.manage` |

**PUT thresholds:**

```json
{
  "rules": [
    {
      "measure_key": "trip_count",
      "op": "GT",
      "value_json": 100,
      "severity": "WARN",
      "subscriptions": [{ "kind": "ROLE", "id": "…" }]
    }
  ]
}
```

Evaluation runs on refresh; fires `dashboard.alert.fired` + optional p15.

---

## 11. Packs & admin

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/dashboard/packages` | `dashboard.admin` |
| `POST` | `/api/v1/dashboard/packages/{package_key}/apply` | `dashboard.admin` |
| `GET` | `/api/v1/dashboard/iframe-allowlist` | `dashboard.admin` |
| `PUT` | `/api/v1/dashboard/iframe-allowlist` | `dashboard.admin` |
| `GET` | `/api/v1/dashboard/admin/budgets` | `dashboard.admin` |
| `PUT` | `/api/v1/dashboard/admin/budgets` | `dashboard.admin` |

Applying a pack clones SYSTEM boards into tenant (or updates if lineage matches) and respects p12/p26 gates.

---

## 12. Permission matrix (summary)

| Surface | Min permission |
|---|---|
| Catalog / runtime / data | `dashboard.catalog.read` |
| Board edit | `dashboard.board.manage` |
| Widgets / layout / bindings | `dashboard.widget.manage` |
| Publish | `dashboard.publish` |
| Personalize / presets | `dashboard.personalize` |
| Shares / home assign | `dashboard.share.manage` |
| Thresholds | `dashboard.alert.manage` |
| Packs / allowlist / budgets | `dashboard.admin` |

---

## 13. Example flows

### 13.1 Open role home

1. Shell calls `GET /dashboard/home`  
2. `GET /boards/{id}/runtime`  
3. `POST /boards/{id}/data` with branch filter  
4. Render KPIs/charts; SWR cache on subsequent polls  

### 13.2 Designer publish

1. Create board + widgets + bindings  
2. `POST .../widgets/{id}/data` preview as self  
3. `POST .../publish` → live version immutable  
4. Assign role home  

### 13.3 Drill to report

1. User clicks KPI  
2. `POST .../drill` → report params  
3. Frontend opens p24 report run UI / API  

---

## 14. Related documents

- Guide: [`DASHBOARD_GUIDE.md`](DASHBOARD_GUIDE.md)  
- Schema: [`DASHBOARD_SCHEMA.md`](DASHBOARD_SCHEMA.md)  
- Reporting: [`../24_reporting/REPORTING_API.md`](../24_reporting/REPORTING_API.md)  
- Feature: [`../12_feature/FEATURE_API.md`](../12_feature/FEATURE_API.md)  
- Configuration: [`../03_configuration/CONFIGURATION_API.md`](../03_configuration/CONFIGURATION_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
