# JeslotERP Reporting Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `reporting_dataset` + `reporting_report` are the HTTP catalog ledger. Warehouse execute stays on the engine port. Not Production.  
**Package:** `platforms.p24_reporting`  
**PostgreSQL schema:** `reporting`  
**Companion:** [`REPORTING_GUIDE.md`](REPORTING_GUIDE.md) · [`REPORTING_API.md`](REPORTING_API.md)

> Runtime models: `platforms/p24_reporting/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `reporting` (never `p24`) |
| Tables | `reporting_*` |
| Soft delete | Archive/retire defs; runs retained per policy |
| Cross-schema | UUID refs (tenant, branch, media, user, schedule) |
| RLS | FORCE on tenant defs, runs, exports, shares, snapshots |
| Secrets | None in reporting; download via media signed URLs |

---

## 2. Complete table inventory (**62 tables**)

### 2.1 Catalog & folders (7)

| # | Table | Purpose |
|---|---|---|
| 1 | `reporting_folder` | Folder tree |
| 2 | `reporting_folder_closure` | Nested set / closure |
| 3 | `reporting_favorite` | User favorites |
| 4 | `reporting_tag` | Tags |
| 5 | `reporting_tag_link` | Tag ↔ report |
| 6 | `reporting_catalog_entry` | Unified catalog projection |
| 7 | `reporting_lifecycle` | Draft/published/retired meta |

### 2.2 Report types & packs (5)

| # | Table | Purpose |
|---|---|---|
| 8 | `reporting_report_type` | Type templates |
| 9 | `reporting_report_type_field` | Allowed fields |
| 10 | `reporting_package` | Packs |
| 11 | `reporting_package_item` | Pack contents |
| 12 | `reporting_feature_binding` | Feature gates |

### 2.3 Datasets & semantic model (12)

| # | Table | Purpose |
|---|---|---|
| 13 | `reporting_dataset` | Datasets |
| 14 | `reporting_dataset_version` | Versioned defs |
| 15 | `reporting_dataset_source` | Source entities/views |
| 16 | `reporting_dataset_join` | Joins |
| 17 | `reporting_dataset_field` | Dimensions/attributes |
| 18 | `reporting_dataset_measure` | Measures |
| 19 | `reporting_dataset_filter` | Default filters |
| 20 | `reporting_dataset_param` | Dataset parameters |
| 21 | `reporting_dataset_rls` | RLS policies |
| 22 | `reporting_dataset_rls_rule` | Rule rows |
| 23 | `reporting_dataset_budget` | Cost budgets |
| 24 | `reporting_dataset_dependency` | Upstream deps |

### 2.4 Report definitions (10)

| # | Table | Purpose |
|---|---|---|
| 25 | `reporting_report` | Report header |
| 26 | `reporting_report_version` | Versioned layout |
| 27 | `reporting_report_column` | Columns |
| 28 | `reporting_report_group` | Groupings |
| 29 | `reporting_report_sort` | Sorts |
| 30 | `reporting_report_filter` | Def-level filters |
| 31 | `reporting_report_formula` | Calculated fields |
| 32 | `reporting_report_format` | Conditional formats |
| 33 | `reporting_report_print_option` | Page/print |
| 34 | `reporting_report_dataset_bind` | Multi-dataset binds |

### 2.5 Parameters & variants (5)

| # | Table | Purpose |
|---|---|---|
| 35 | `reporting_parameter` | Parameter defs |
| 36 | `reporting_parameter_option` | Enum/options |
| 37 | `reporting_variant` | Saved variants |
| 38 | `reporting_variant_value` | Param values |
| 39 | `reporting_variant_column` | Column visibility overrides |

### 2.6 Execution (7)

| # | Table | Purpose |
|---|---|---|
| 40 | `reporting_run` | Execution runs |
| 41 | `reporting_run_parameter` | Bound params |
| 42 | `reporting_run_page` | Cached page refs / cursors |
| 43 | `reporting_run_metric` | Duration, rows, cost |
| 44 | `reporting_run_error` | Structured errors |
| 45 | `reporting_execution_plan` | Compiled plan checksum |
| 46 | `reporting_cancel_request` | Cancel signals |

### 2.7 Exports & delivery (6)

| # | Table | Purpose |
|---|---|---|
| 47 | `reporting_export` | Export jobs |
| 48 | `reporting_export_option` | Format options |
| 49 | `reporting_artifact` | media_id + checksum |
| 50 | `reporting_delivery` | Delivery attempts |
| 51 | `reporting_delivery_recipient` | Recipients |
| 52 | `reporting_download_grant` | Time-bound grants |

### 2.8 Schedules & subscriptions (5)

| # | Table | Purpose |
|---|---|---|
| 53 | `reporting_subscription` | Subscriptions |
| 54 | `reporting_subscription_recipient` | Users/emails/roles |
| 55 | `reporting_burst_rule` | Burst filters |
| 56 | `reporting_schedule_binding` | Link to p17 schedule_id |
| 57 | `reporting_subscription_run` | Fire history / idempotency |

### 2.9 Sharing, snapshots, governance (5)

| # | Table | Purpose |
|---|---|---|
| 58 | `reporting_share` | ACL shares |
| 59 | `reporting_snapshot` | Sealed snapshots |
| 60 | `reporting_snapshot_row_meta` | Row count / partition meta |
| 61 | `reporting_changeset` | Definition changesets |
| 62 | `reporting_approval` | Publish approvals |

**Plumbing:** `reporting_outbox`, `reporting_idempotency_key`

**Implementation total with plumbing: 64 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `reporting_lifecycle` | `DRAFT`, `PUBLISHED`, `DEPRECATED`, `RETIRED` |
| `reporting_layout_kind` | `TABLE`, `MATRIX`, `GROUPED`, `STATEMENT`, `LIST` |
| `reporting_field_role` | `DIMENSION`, `ATTRIBUTE`, `MEASURE`, `FILTER_ONLY` |
| `reporting_agg` | `SUM`, `COUNT`, `COUNT_DISTINCT`, `AVG`, `MIN`, `MAX` |
| `reporting_run_status` | `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `TIMED_OUT` |
| `reporting_export_format` | `CSV`, `XLSX`, `PDF`, `JSON`, `PARQUET` |
| `reporting_share_scope` | `PRIVATE`, `TENANT`, `ROLE`, `USER`, `LINK` |
| `reporting_param_type` | `STRING`, `INT`, `DECIMAL`, `DATE`, `DATETIME`, `BOOL`, `UUID`, `ENUM`, `BRANCH`, `ENTITY_REF` |

---

## 4. Dataset detail

### 4.1 `reporting_dataset`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | NULL = system pack |
| `dataset_key` | VARCHAR(100) | Unique per tenant/system |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `lifecycle` | VARCHAR(20) | |
| `primary_entity_key` | VARCHAR(100) NULL | p05 entity |
| `published_version_id` | UUID NULL | |

### 4.2 `reporting_dataset_version`

| Column | Type | Notes |
|---|---|---|
| `dataset_id` | UUID | |
| `version` | INT | |
| `checksum` | VARCHAR(64) | |
| `compiled` | JSONB | Intermediate IR |
| `published_at` | TIMESTAMPTZ NULL | |

### 4.3 `reporting_dataset_field`

| Column | Type | Notes |
|---|---|---|
| `dataset_version_id` | UUID | |
| `field_key` | VARCHAR(100) | |
| `label_key` | VARCHAR(150) NULL | p06 |
| `data_type` | VARCHAR(30) | |
| `role` | VARCHAR(20) | |
| `source_path` | VARCHAR(300) | Entity.field / expression |
| `is_sensitive` | BOOLEAN | Masking hint |
| `metadata_field_id` | UUID NULL | Soft ref p05 |

### 4.4 `reporting_dataset_measure`

| Column | Type | Notes |
|---|---|---|
| `measure_key` | VARCHAR(100) | |
| `agg` | VARCHAR(30) | |
| `expression` | TEXT | Approved expression DSL |
| `grain` | VARCHAR(100) NULL | |

### 4.5 `reporting_dataset_rls` / `_rule`

Policy key + rules: claim path (`tenant_id`, `branch_ids`, `partner_id`) → column predicate.  
Deny by default when no matching allow rule for non-admin.

### 4.6 `reporting_dataset_budget`

| Column | Type | Notes |
|---|---|---|
| `max_rows` | INT | |
| `max_runtime_ms` | INT | |
| `max_export_bytes` | BIGINT | |
| `sync_max_rows` | INT | Interactive cap |

---

## 5. Report definition detail

### 5.1 `reporting_report`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | |
| `folder_id` | UUID NULL | |
| `report_key` | VARCHAR(100) | |
| `name` | VARCHAR(150) | |
| `report_type_id` | UUID NULL | |
| `layout_kind` | VARCHAR(20) | |
| `lifecycle` | VARCHAR(20) | |
| `owner_user_id` | UUID NULL | |
| `published_version_id` | UUID NULL | |

### 5.2 `reporting_report_version`

| Column | Type | Notes |
|---|---|---|
| `report_id` | UUID | |
| `version` | INT | |
| `dataset_id` | UUID | Primary dataset |
| `checksum` | VARCHAR(64) | |
| `layout` | JSONB | Full layout blob optional |
| `change_notes` | TEXT NULL | |

### 5.3 `reporting_report_column`

| Column | Type | Notes |
|---|---|---|
| `report_version_id` | UUID | |
| `field_key` / `measure_key` | VARCHAR | |
| `ordinal` | INT | |
| `width` | INT NULL | |
| `label_override_key` | VARCHAR(150) NULL | |
| `visibility` | VARCHAR(20) | `ALWAYS`, `OPTIONAL`, `HIDDEN` |

---

## 6. Parameters & variants

### 6.1 `reporting_parameter`

| Column | Type | Notes |
|---|---|---|
| `owner_kind` | VARCHAR(20) | `DATASET`, `REPORT` |
| `owner_id` | UUID | |
| `param_key` | VARCHAR(80) | |
| `param_type` | VARCHAR(30) | |
| `is_required` | BOOLEAN | |
| `default_json` | JSONB NULL | |
| `label_key` | VARCHAR(150) NULL | |

### 6.2 `reporting_variant`

Named saved run config: report_id, user_id, name, is_shared.

---

## 7. Execution & export

### 7.1 `reporting_run`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | RLS |
| `report_id` | UUID | |
| `report_version_id` | UUID | |
| `variant_id` | UUID NULL | |
| `status` | VARCHAR(20) | |
| `triggered_by` | VARCHAR(20) | `USER`, `SCHEDULE`, `API`, `DASHBOARD` |
| `actor_user_id` | UUID NULL | |
| `job_id` | UUID NULL | p14 soft ref |
| `plan_checksum` | VARCHAR(64) | |
| `row_count` | BIGINT NULL | |
| `started_at` / `finished_at` | TIMESTAMPTZ | |

Result payloads: short interactive pages in cache; large results streamed to export/snapshot storage — **not** unbounded JSON in PG.

### 7.2 `reporting_export`

| Column | Type | Notes |
|---|---|---|
| `run_id` | UUID | |
| `format` | VARCHAR(20) | |
| `status` | VARCHAR(20) | |
| `locale` | VARCHAR(20) NULL | |
| `artifact_id` | UUID NULL | |

### 7.3 `reporting_artifact`

| Column | Type | Notes |
|---|---|---|
| `media_id` | UUID | p08 |
| `checksum` | VARCHAR(64) | |
| `byte_size` | BIGINT | |
| `content_type` | VARCHAR(100) | |

---

## 8. Subscriptions

### 8.1 `reporting_subscription`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `report_id` | UUID | |
| `variant_id` | UUID NULL | |
| `format` | VARCHAR(20) | |
| `is_active` | BOOLEAN | |
| `timezone` | VARCHAR(64) | |

### 8.2 `reporting_schedule_binding`

| Column | Type | Notes |
|---|---|---|
| `subscription_id` | UUID | |
| `scheduler_job_id` | UUID | Soft ref p17 |
| `idempotency_window_key` | VARCHAR(100) | Prevents double fire |

### 8.3 `reporting_burst_rule`

Slice expression (e.g. per `branch_id`) generating N deliveries per fire.

---

## 9. Sharing & snapshots

### 9.1 `reporting_share`

| Column | Type | Notes |
|---|---|---|
| `resource_kind` | VARCHAR(20) | `REPORT`, `FOLDER`, `DATASET`, `SNAPSHOT` |
| `resource_id` | UUID | |
| `scope` | VARCHAR(20) | |
| `principal_id` | UUID NULL | User/role |
| `can_run` / `can_export` / `can_edit` | BOOLEAN | |
| `expires_at` | TIMESTAMPTZ NULL | |

### 9.2 `reporting_snapshot`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `report_id` | UUID | |
| `period_key` | VARCHAR(40) | `2026-08` |
| `sealed_at` | TIMESTAMPTZ | |
| `seal_checksum` | VARCHAR(64) | |
| `artifact_id` | UUID NULL | Optional full dump |
| `is_immutable` | BOOLEAN | Always true after seal |

---

## 10. Governance & packs

- Packs: `freight.ops.v1`, `finance.gst.v1`, `fleet.efficiency.v1`  
- Changesets + approvals for publish in regulated tenants  
- Feature bindings for `PARQUET` / premium PDF  

---

## 11. Plumbing

| Table | Purpose |
|---|---|
| `reporting_outbox` | Domain events |
| `reporting_idempotency_key` | Admin + schedule fires |

---

## 12. RLS summary

| Class | Policy |
|---|---|
| System packs | Read authenticated; manage admin |
| Tenant datasets/reports | FORCE `tenant_id` + share ACL |
| Runs/exports/artifacts meta | FORCE tenant + actor/share |
| Snapshots | FORCE tenant; sealed no update |

---

## 13. Seed minimum

1. Folders: Company / Operations / Finance / Fleet  
2. Datasets stubs aligned to live domains (identity users count demo; org companies)  
3. Sample TABLE report + variant  
4. Budgets: sync 5k rows / 10s; async 1M / 5min  
5. Permissions `reporting.*`  
6. Export formats CSV/XLSX/PDF enabled  
7. Package `core.samples.v1`  

---

## 14. ER overview

```text
folder ── reports ── versions ── columns/groups/filters
              │
           dataset ── versions ── fields/measures/joins/rls
              │
           parameters ── variants
              │
           runs ── exports ── artifacts (media_id)
              │
           subscriptions ── schedule_binding / burst
              │
           shares / snapshots / packages
```

---

## 15. Implementation notes

1. Expression DSL must be sandboxed (no raw SQL injection).  
2. Compile checksum ties report version + dataset version + params schema.  
3. Cancel cooperates with p14 job cancellation.  
4. Split models: `catalog`, `dataset`, `report`, `param`, `execution`, `export`, `subscription`, `security`, `governance`, `plumbing`.
