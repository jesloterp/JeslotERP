# JeslotERP Dashboard Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `dashboard_folder` / `dashboard_board` / `dashboard_widget` are the HTTP board ledger. Widget samples stay memory. Not Production.  
**Package:** `platforms.p25_dashboard`  
**PostgreSQL schema:** `dashboard`  
**Companion:** [`DASHBOARD_GUIDE.md`](DASHBOARD_GUIDE.md) · [`DASHBOARD_API.md`](DASHBOARD_API.md)

> Runtime models: `platforms/p25_dashboard/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `dashboard` (never `p25`) |
| Tables | `dashboard_*` |
| Soft delete | Retire boards; keep versions |
| Cross-schema | UUID refs (tenant, user, role, dataset, report, media) |
| RLS | FORCE on tenant boards, shares, personalizations, alerts |
| Secrets | None; iframe allowlist policy only |

---

## 2. Complete table inventory (**60 tables**)

### 2.1 Catalog & folders (6)

| # | Table | Purpose |
|---|---|---|
| 1 | `dashboard_folder` | Folder tree |
| 2 | `dashboard_folder_closure` | Hierarchy |
| 3 | `dashboard_favorite` | User favorites |
| 4 | `dashboard_tag` | Tags |
| 5 | `dashboard_tag_link` | Tag ↔ board |
| 6 | `dashboard_catalog_entry` | Projection |

### 2.2 Boards & versions (7)

| # | Table | Purpose |
|---|---|---|
| 7 | `dashboard_board` | Board header |
| 8 | `dashboard_board_version` | Versioned defs |
| 9 | `dashboard_board_locale` | Title/desc i18n keys |
| 10 | `dashboard_board_default_filter` | Default filter bar |
| 11 | `dashboard_board_theme` | Density/theme hints |
| 12 | `dashboard_lifecycle` | Lifecycle meta |
| 13 | `dashboard_clone_lineage` | Pack → tenant fork |

### 2.3 Layouts (6)

| # | Table | Purpose |
|---|---|---|
| 14 | `dashboard_layout` | Layout root |
| 15 | `dashboard_layout_section` | Sections |
| 16 | `dashboard_layout_breakpoint` | sm/md/lg grids |
| 17 | `dashboard_grid_cell` | Cell placements |
| 18 | `dashboard_layout_guide` | Snap guides |
| 19 | `dashboard_layout_lock` | Lock regions |

### 2.4 Widget registry & instances (10)

| # | Table | Purpose |
|---|---|---|
| 20 | `dashboard_widget_type` | Type registry |
| 21 | `dashboard_widget_type_schema` | Config JSON schema |
| 22 | `dashboard_widget` | Widget instances |
| 23 | `dashboard_widget_config` | Config blob |
| 24 | `dashboard_widget_placement` | x,y,w,h per breakpoint |
| 25 | `dashboard_widget_title` | Titles / label keys |
| 26 | `dashboard_widget_visibility` | Role/feature visibility |
| 27 | `dashboard_widget_action` | Toolbar actions |
| 28 | `dashboard_feature_binding` | Feature gates |
| 29 | `dashboard_iframe_allowlist` | Allowed iframe hosts |

### 2.5 Bindings & filters (8)

| # | Table | Purpose |
|---|---|---|
| 30 | `dashboard_binding` | Dataset/report binds |
| 31 | `dashboard_binding_field` | Projected fields |
| 32 | `dashboard_binding_measure` | Measures |
| 33 | `dashboard_filter_def` | Filter definitions |
| 34 | `dashboard_filter_option` | Enum options |
| 35 | `dashboard_widget_filter` | Widget filter links |
| 36 | `dashboard_filter_sync` | Sync groups |
| 37 | `dashboard_query_budget` | Per-widget budgets |

### 2.6 Refresh & cache (5)

| # | Table | Purpose |
|---|---|---|
| 38 | `dashboard_refresh_policy` | Refresh policies |
| 39 | `dashboard_widget_refresh` | Per-widget policy |
| 40 | `dashboard_cache_key_template` | Cache key templates |
| 41 | `dashboard_refresh_job` | Warm jobs meta |
| 42 | `dashboard_stale_policy` | SWR TTLs |

### 2.7 Personalization & home (7)

| # | Table | Purpose |
|---|---|---|
| 43 | `dashboard_personalization` | User board overrides |
| 44 | `dashboard_personal_widget` | Hide/move widgets |
| 45 | `dashboard_filter_preset` | Saved filter presets |
| 46 | `dashboard_home_assignment` | Role/user → board |
| 47 | `dashboard_user_preference` | Shell prefs |
| 48 | `dashboard_recent_board` | Recents |
| 49 | `dashboard_pin` | Pinned boards |

### 2.8 Sharing, drill, alerts (7)

| # | Table | Purpose |
|---|---|---|
| 50 | `dashboard_share` | ACL |
| 51 | `dashboard_drill_target` | Drill contracts |
| 52 | `dashboard_threshold_rule` | Alert rules |
| 53 | `dashboard_threshold_state` | Current breach state |
| 54 | `dashboard_alert_subscription` | Who gets notified |
| 55 | `dashboard_alert_event` | Fired history |
| 56 | `dashboard_link_share` | Time-bound links |

### 2.9 Governance (4)

| # | Table | Purpose |
|---|---|---|
| 57 | `dashboard_package` | Packs |
| 58 | `dashboard_package_item` | Items |
| 59 | `dashboard_changeset` | Changesets |
| 60 | `dashboard_approval` | Publish approvals |

**Plumbing:** `dashboard_outbox`, `dashboard_idempotency_key`

**Implementation total with plumbing: 62 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `dashboard_lifecycle` | `DRAFT`, `PUBLISHED`, `DEPRECATED`, `RETIRED` |
| `dashboard_scope` | `SYSTEM`, `TENANT`, `PERSONAL` |
| `dashboard_widget_kind` | `KPI`, `CHART`, `TABLE`, `LIST`, `FILTER`, `REPORT_EMBED`, `MARKDOWN`, `IFRAME`, `SPACER` |
| `dashboard_chart_type` | `LINE`, `BAR`, `AREA`, `PIE`, `DONUT`, `STACKED_BAR` |
| `dashboard_refresh_mode` | `MANUAL`, `ON_OPEN`, `INTERVAL`, `REALTIME`, `EVENT` |
| `dashboard_share_scope` | `PRIVATE`, `TENANT`, `ROLE`, `USER`, `LINK` |
| `dashboard_drill_kind` | `REPORT`, `ENTITY_ROUTE`, `BOARD`, `EXTERNAL_URL` |
| `dashboard_threshold_op` | `GT`, `GTE`, `LT`, `LTE`, `EQ`, `BETWEEN` |

---

## 4. Board & layout detail

### 4.1 `dashboard_board`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | NULL for SYSTEM packs |
| `board_key` | VARCHAR(100) | Unique per scope |
| `name` | VARCHAR(150) | |
| `scope` | VARCHAR(20) | |
| `folder_id` | UUID NULL | |
| `lifecycle` | VARCHAR(20) | |
| `owner_user_id` | UUID NULL | |
| `published_version_id` | UUID NULL | |
| `is_home_eligible` | BOOLEAN | |

### 4.2 `dashboard_board_version`

| Column | Type | Notes |
|---|---|---|
| `board_id` | UUID | |
| `version` | INT | |
| `checksum` | VARCHAR(64) | |
| `layout_id` | UUID | |
| `change_notes` | TEXT NULL | |
| `published_at` | TIMESTAMPTZ NULL | |

### 4.3 `dashboard_layout_section`

| Column | Type | Notes |
|---|---|---|
| `layout_id` | UUID | |
| `section_key` | VARCHAR(80) | |
| `title_key` | VARCHAR(150) NULL | p06 |
| `ordinal` | INT | |
| `columns` | INT | Default 12 |

### 4.4 `dashboard_widget_placement`

| Column | Type | Notes |
|---|---|---|
| `widget_id` | UUID | |
| `breakpoint` | VARCHAR(10) | `lg`, `md`, `sm` |
| `x` / `y` / `w` / `h` | INT | Grid units |

---

## 5. Widgets & bindings

### 5.1 `dashboard_widget_type`

| Column | Type | Notes |
|---|---|---|
| `type_key` | VARCHAR(50) UNIQUE | `kpi.v1` |
| `kind` | VARCHAR(30) | |
| `is_system` | BOOLEAN | |
| `feature_flag_key` | VARCHAR(150) NULL | |

### 5.2 `dashboard_widget`

| Column | Type | Notes |
|---|---|---|
| `board_version_id` | UUID | |
| `widget_key` | VARCHAR(80) | |
| `type_id` | UUID | |
| `section_id` | UUID NULL | |
| `refresh_policy_id` | UUID NULL | |
| `is_locked` | BOOLEAN | Personalization lock |

### 5.3 `dashboard_binding`

| Column | Type | Notes |
|---|---|---|
| `widget_id` | UUID | |
| `dataset_id` | UUID NULL | Soft p24 |
| `report_id` | UUID NULL | Soft p24 |
| `binding_mode` | VARCHAR(20) | `DATASET`, `REPORT_SLICE`, `STATIC` |
| `param_map` | JSONB | Board filter → dataset params |
| `limit_rows` | INT NULL | |

### 5.4 `dashboard_binding_measure`

| Column | Type | Notes |
|---|---|---|
| `binding_id` | UUID | |
| `measure_key` | VARCHAR(100) | From p24 contract |
| `alias` | VARCHAR(80) NULL | |
| `format` | VARCHAR(40) NULL | `currency`, `percent` |

---

## 6. Filters

### 6.1 `dashboard_filter_def`

| Column | Type | Notes |
|---|---|---|
| `board_version_id` | UUID | |
| `filter_key` | VARCHAR(80) | |
| `param_type` | VARCHAR(30) | Align reporting param types |
| `label_key` | VARCHAR(150) NULL | |
| `is_global` | BOOLEAN | Filter bar |
| `default_json` | JSONB NULL | |

### 6.2 `dashboard_filter_sync`

Groups filters that stay in sync across widgets (`sync_group_key`).

---

## 7. Refresh & cache

### 7.1 `dashboard_refresh_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | |
| `mode` | VARCHAR(20) | |
| `interval_seconds` | INT NULL | |
| `event_types` | JSONB NULL | p13 types for EVENT mode |

### 7.2 `dashboard_stale_policy`

| Column | Type | Notes |
|---|---|---|
| `ttl_seconds` | INT | |
| `swr_seconds` | INT | Serve stale while refresh |
| `max_age_seconds` | INT | Hard expire |

Cache payload lives in p16/Redis — PG stores templates/jobs only.

---

## 8. Personalization & home

### 8.1 `dashboard_personalization`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `board_id` | UUID | |
| `user_id` | UUID | |
| `allow_reorder` | BOOLEAN | Copied from board policy at save |
| `filter_preset_id` | UUID NULL | |

### 8.2 `dashboard_home_assignment`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `principal_kind` | VARCHAR(20) | `ROLE`, `USER` |
| `principal_id` | UUID | |
| `board_id` | UUID | |
| `priority` | INT | User > role |

---

## 9. Sharing, drill, alerts

### 9.1 `dashboard_share`

| Column | Type | Notes |
|---|---|---|
| `board_id` | UUID | |
| `scope` | VARCHAR(20) | |
| `principal_id` | UUID NULL | |
| `can_view` / `can_edit` / `can_share` | BOOLEAN | |
| `expires_at` | TIMESTAMPTZ NULL | |

### 9.2 `dashboard_drill_target`

| Column | Type | Notes |
|---|---|---|
| `widget_id` | UUID | |
| `drill_kind` | VARCHAR(20) | |
| `target_ref` | VARCHAR(300) | report_id / route template |
| `param_map` | JSONB | |

### 9.3 `dashboard_threshold_rule`

| Column | Type | Notes |
|---|---|---|
| `widget_id` | UUID | |
| `measure_key` | VARCHAR(100) | |
| `op` | VARCHAR(10) | |
| `value_json` | JSONB | |
| `severity` | VARCHAR(20) | `INFO`, `WARN`, `CRITICAL` |
| `is_active` | BOOLEAN | |

---

## 10. Governance & packs

- Packs: `ops.dispatch.home.v1`, `finance.overview.v1`, `fleet.efficiency.v1`  
- Clone lineage tracks pack item → tenant board  
- Approvals optional for regulated publish  

---

## 11. Plumbing

| Table | Purpose |
|---|---|
| `dashboard_outbox` | Domain events |
| `dashboard_idempotency_key` | Admin APIs |

---

## 12. RLS summary

| Class | Policy |
|---|---|
| SYSTEM boards | Read if published; manage admin |
| TENANT boards | FORCE `tenant_id` + share |
| PERSONAL | Owner user only (+ admin) |
| Personalizations / presets / alerts | FORCE tenant + user |

---

## 13. Seed minimum

1. Widget types: KPI, CHART, TABLE, FILTER, MARKDOWN  
2. Refresh policies: ON_OPEN, INTERVAL_60, MANUAL  
3. Sample SYSTEM board `ops.sample.home` (draft until pack apply)  
4. Permissions `dashboard.*`  
5. Query budgets: KPI 1 row aggregate; table 100 rows sync  
6. Package `core.samples.v1`  

---

## 14. ER overview

```text
folder ── boards ── versions ── layout ── sections
                         │
                      widgets ── placement / config
                         │
                      bindings ── fields/measures (→ p24)
                         │
                      filters / refresh / drill / thresholds
personalization / home_assignment / shares
packages
```

---

## 15. Implementation notes

1. Resolve home: highest priority USER assignment else ROLE else tenant default else system.  
2. Widget data API composes filters then calls p24 gateway; never queries domain tables directly.  
3. Split models: `catalog`, `board`, `layout`, `widget`, `binding`, `refresh`, `personalize`, `security`, `governance`, `plumbing`.
