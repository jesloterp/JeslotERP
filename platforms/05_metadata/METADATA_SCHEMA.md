# JeslotERP Metadata Platform — Production Schema (Advanced)

**Version:** 2.0  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** (ORM + migrations `e7f8a9b0c1d2`, `f8a9b0c1d2e3`; HTTP dictionary/overlay/publish persist on AsyncSession)  
**Package:** `platforms.p05_metadata`  
**PostgreSQL schema:** `metadata`  
**Companion:** [`METADATA_GUIDE.md`](METADATA_GUIDE.md) · [`METADATA_API.md`](METADATA_API.md)

> Runtime models: `platforms/p05_metadata/infrastructure/persistence/models/`.  
> This document is authoritative for DDL until code exists.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `metadata` |
| Tables | `metadata_*` |
| PK | UUID `gen_random_uuid()` |
| Soft delete | `is_deleted` + partial uniques |
| Keys | Immutable after first publish |
| Cross-schema | UUID refs only |
| RLS | FORCE on tenant-scoped tables |
| Artifacts | Immutable; content checksum |
| Status | `ACTIVE` \| `INACTIVE` \| `DEPRECATED` \| `DELETED` |
| Lifecycle | `DRAFT` \| `IN_REVIEW` \| `PUBLISHED` \| `ARCHIVED` |

---

## 2. Complete table inventory (**52 tables**)

### 2.1 System lookups (8)

| # | Table | Purpose |
|---|---|---|
| 1 | `metadata_data_type` | Physical types |
| 2 | `metadata_semantic_type` | Business semantic types |
| 3 | `metadata_ui_control` | Controls + a11y defaults |
| 4 | `metadata_validation_type` | Rule kinds |
| 5 | `metadata_relation_kind` | Relation cardinalities |
| 6 | `metadata_storage_strategy` | Persistence strategies |
| 7 | `metadata_channel` | WEB_DENSE, MOBILE, … |
| 8 | `metadata_classification` | PUBLIC…SPI classification |

### 2.2 Dictionary core (9)

| # | Table | Purpose |
|---|---|---|
| 9 | `metadata_module` | Module/namespace |
| 10 | `metadata_business_domain` | Semantic domain (Party, Finance, …) |
| 11 | `metadata_entity` | Entity descriptors |
| 12 | `metadata_field` | Field descriptors |
| 13 | `metadata_field_option` | Enum options |
| 14 | `metadata_relation` | Relations |
| 15 | `metadata_entity_index` | Index docs |
| 16 | `metadata_entity_constraint` | Constraint docs |
| 17 | `metadata_composite_field` | COMPOSITE sub-structure |

### 2.3 Semantic / analytics (4)

| # | Table | Purpose |
|---|---|---|
| 18 | `metadata_measure` | Report measures |
| 19 | `metadata_dimension` | Report dimensions |
| 20 | `metadata_metric_binding` | Measure↔field/entity |
| 21 | `metadata_search_analyzer` | Search analyzer hints per field/semantic |

### 2.4 Behavior (4)

| # | Table | Purpose |
|---|---|---|
| 22 | `metadata_validation_rule` | Declarative validations |
| 23 | `metadata_expression` | Named safe ASTs |
| 24 | `metadata_field_default` | Default AST/value |
| 25 | `metadata_computed_field` | Virtual compute AST |

### 2.5 Security descriptors (3)

| # | Table | Purpose |
|---|---|---|
| 26 | `metadata_field_security` | FLS / mask / perms |
| 27 | `metadata_entity_security` | Entity default perms |
| 28 | `metadata_masking_policy` | Named mask strategies |

### 2.6 UI descriptors (12)

| # | Table | Purpose |
|---|---|---|
| 29 | `metadata_form` | Forms |
| 30 | `metadata_form_section` | Sections |
| 31 | `metadata_form_field` | Field placements + ASTs |
| 32 | `metadata_form_variant` | Role/channel variant map |
| 33 | `metadata_list_view` | Grids |
| 34 | `metadata_list_column` | Columns |
| 35 | `metadata_list_variant` | Role/channel variants |
| 36 | `metadata_filter` | Filters |
| 37 | `metadata_filter_field` | Filter fields |
| 38 | `metadata_action` | Actions |
| 39 | `metadata_action_variant` | Conditional actions |
| 40 | `metadata_inspector` | Side inspector layouts |

### 2.7 Extensibility & packs (5)

| # | Table | Purpose |
|---|---|---|
| 41 | `metadata_entity_overlay` | Tenant/company overlays |
| 42 | `metadata_extension_value` | EAV values |
| 43 | `metadata_user_preference` | User column order/hidden |
| 44 | `metadata_package` | Installable packs |
| 45 | `metadata_package_item` | Pack contents |

### 2.8 Governance / publish (7)

| # | Table | Purpose |
|---|---|---|
| 46 | `metadata_changeset` | Change batches |
| 47 | `metadata_changeset_item` | Ops |
| 48 | `metadata_approval` | Approvals |
| 49 | `metadata_publish_version` | Versions |
| 50 | `metadata_publish_artifact` | Snapshots |
| 51 | `metadata_dependency_edge` | Graph edges |
| 52 | `metadata_catalog_audit` | Catalog audit trail |

### 2.9 Interop + quality + plumbing (8) — **extended set**

| # | Table | Purpose |
|---|---|---|
| 53 | `metadata_command_descriptor` | Command catalog |
| 54 | `metadata_query_descriptor` | Query catalog |
| 55 | `metadata_event_descriptor` | Event catalog |
| 56 | `metadata_drift_report` | Drift scan header |
| 57 | `metadata_drift_finding` | Drift rows |
| 58 | `metadata_outbox` | Outbox |
| 59 | `metadata_idempotency_key` | Idempotency |
| 60 | `metadata_resolve_cache` | Optional durable cache entries |

**Contract total: 60 tables** (advanced enterprise surface).

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `metadata_channel_code` | `WEB_DENSE`, `WEB_COMFORT`, `MOBILE`, `TABLET`, `PRINT`, `ACCESSIBLE` |
| `metadata_classification_code` | `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED`, `PII`, `SPI` |
| `metadata_mask_strategy` | `NONE`, `LAST4`, `FIRST_LAST`, `HASH`, `REDACT`, `TOKENIZE` |
| `metadata_dependency_type` | `FIELD_USES_ENTITY`, `FORM_USES_FIELD`, `ACTION_USES_PERMISSION`, `VALIDATION_USES_FIELD`, `MEASURE_USES_FIELD`, `EVENT_USES_ENTITY`, `PACK_USES_ENTITY` |
| `metadata_drift_status` | `IN_SYNC`, `MISSING_IN_DB`, `MISSING_IN_METADATA`, `TYPE_MISMATCH`, `NULLABILITY_MISMATCH`, `EXTRA_IN_DB` |
| `metadata_approval_status` | `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED` |
| `metadata_changeset_op` | `CREATE`, `UPDATE`, `DEPRECATE`, `RESTORE`, `OVERLAY_SET`, `SECURITY_SET` |
| `metadata_expression_purpose` | `VISIBILITY`, `REQUIRED`, `READONLY`, `DEFAULT`, `COMPUTE`, `FILTER` |

Physical + semantic type codes: see Guide §13.

---

## 4. Shared mixin columns

Standard audit/lifecycle/OCC/extension bags as in v1, plus optional:

| Column | Type | Notes |
|---|---|---|
| `origin_layer` | VARCHAR(30) NULL | Set on effective projections only (not always persisted) |
| `content_checksum` | VARCHAR(64) NULL | For artifacts/packs |
| `feature_flag_key` | VARCHAR(100) NULL | Gate inclusion |

---

## 5. Lookups (additions beyond v1)

### 5.1 `metadata_semantic_type`

| Column | Type | Notes |
|---|---|---|
| `code` | VARCHAR(50) UNIQUE | `GSTIN`, `PAN`, … |
| `name` | VARCHAR(150) | |
| `base_data_type_code` | VARCHAR(30) | Physical base |
| `default_ui_control_code` | VARCHAR(50) | |
| `default_validation_type_code` | VARCHAR(50) | |
| `default_mask_strategy` | VARCHAR(30) | |
| `params_schema` | JSONB | |

### 5.2 `metadata_channel`

| Column | Type | Notes |
|---|---|---|
| `code` | VARCHAR(30) UNIQUE | |
| `name` | VARCHAR(100) | |
| `density` | VARCHAR(20) | `COMPACT`/`COMFORTABLE` |

### 5.3 `metadata_classification`

| Column | Type | Notes |
|---|---|---|
| `code` | VARCHAR(30) UNIQUE | |
| `rank` | INT | Higher = more sensitive |
| `default_mask_strategy` | VARCHAR(30) | |

### 5.4 `metadata_masking_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(100) | |
| `strategy` | VARCHAR(30) | |
| `params` | JSONB | e.g. `{ "visible_last": 4 }` |
| `tenant_id` | UUID NULL | System or tenant |

---

## 6. Dictionary core (enhanced fields)

### 6.1 `metadata_entity` (additions)

| Column | Type | Notes |
|---|---|---|
| `business_domain_id` | UUID NULL | |
| `aggregate_root_key` | VARCHAR(150) NULL | If child entity |
| `concurrency_mode` | VARCHAR(20) | `NONE`/`VERSION` |
| `archive_mode` | VARCHAR(20) | `SOFT`/`HARD` |
| `openapi_tag` | VARCHAR(100) | |
| `icon` | VARCHAR(50) | |
| `color` | VARCHAR(30) | |
| `feature_flag_key` | VARCHAR(100) | |

### 6.2 `metadata_field` (additions)

| Column | Type | Notes |
|---|---|---|
| `semantic_type_code` | VARCHAR(50) NULL | |
| `classification_code` | VARCHAR(30) NULL | Denorm / default |
| `is_computed` | BOOLEAN DEFAULT false | |
| `is_measure_eligible` | BOOLEAN DEFAULT false | |
| `is_dimension_eligible` | BOOLEAN DEFAULT false | |
| `indexing_hint` | VARCHAR(30) NULL | `NONE`/`BTREE`/`GIN`/`TRIGRAM` |
| `analyzer_key` | VARCHAR(50) NULL | Search |
| `feature_flag_key` | VARCHAR(100) NULL | |
| `deprecated_replacement_field_key` | VARCHAR(100) NULL | |

### 6.3 `metadata_composite_field`

For `COMPOSITE` types (address block, money+currency):

| Column | Type | Notes |
|---|---|---|
| `parent_field_id` | UUID | |
| `child_field_key` | VARCHAR(100) | |
| `sort_order` | INT | |

### 6.4 `metadata_business_domain`

| Column | Type | Notes |
|---|---|---|
| `domain_key` | VARCHAR(100) | `party`, `logistics`, `finance` |
| `name` | VARCHAR(150) | |
| `description` | TEXT | |

---

## 7. Semantic / analytics tables

### 7.1 `metadata_measure`

| Column | Type | Notes |
|---|---|---|
| `measure_key` | VARCHAR(150) | |
| `name` | VARCHAR(150) | |
| `entity_id` | UUID NULL | |
| `field_id` | UUID NULL | |
| `aggregation` | VARCHAR(20) | `SUM`/`AVG`/`COUNT`/`MIN`/`MAX` |
| `expression_id` | UUID NULL | Optional AST |
| `tenant_id` | UUID NULL | |

### 7.2 `metadata_dimension`

| Column | Type | Notes |
|---|---|---|
| `dimension_key` | VARCHAR(150) | |
| `entity_id` | UUID | |
| `field_id` | UUID | |
| `hierarchy_rank` | INT NULL | |

### 7.3 `metadata_search_analyzer`

| Column | Type | Notes |
|---|---|---|
| `analyzer_key` | VARCHAR(50) | `standard`, `edge_ngram`, `keyword` |
| `params` | JSONB | |

---

## 8. Behavior tables

### 8.1 `metadata_expression`

| Column | Type | Notes |
|---|---|---|
| `expression_key` | VARCHAR(150) | |
| `purpose` | VARCHAR(30) | |
| `ast` | JSONB NOT NULL | Safe AST document |
| `ast_version` | INT NOT NULL DEFAULT 1 | |
| `tenant_id` | UUID NULL | |
| `checksum` | VARCHAR(64) | |

### 8.2 `metadata_field_default` / `metadata_computed_field`

Bind `field_id` → `expression_id` (+ optional static `default_value` JSONB).

### 8.3 `metadata_validation_rule` (additions)

| Column | Type | Notes |
|---|---|---|
| `expression_id` | UUID NULL | When type = CUSTOM_AST |
| `applies_to_modes` | JSONB | `["CREATE","UPDATE"]` |
| `feature_flag_key` | VARCHAR(100) NULL | |

---

## 9. Security descriptor tables

### 9.1 `metadata_field_security`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | System default or tenant override |
| `field_id` | UUID NOT NULL | |
| `classification_code` | VARCHAR(30) NOT NULL | |
| `mask_policy_id` | UUID NULL | |
| `read_permission` | VARCHAR(100) NULL | |
| `write_permission` | VARCHAR(100) NULL | |
| `reveal_permission` | VARCHAR(100) NULL | Unmask |
| `deny_export` | BOOLEAN DEFAULT false | |

**Unique:** active `(COALESCE(tenant_id,zero), field_id)`.

### 9.2 `metadata_entity_security`

Default `read_permission` / `create_permission` / `update_permission` / `delete_permission` hints for UI packs.

---

## 10. UI tables (additions)

### 10.1 `metadata_form_field` AST columns

| Column | Type |
|---|---|
| `visibility_expression_id` | UUID NULL |
| `required_expression_id` | UUID NULL |
| `readonly_expression_id` | UUID NULL |

### 10.2 `metadata_form_variant` / `list_variant` / `action_variant`

| Column | Type | Notes |
|---|---|---|
| `base_form_id` / list / action | UUID | |
| `channel_code` | VARCHAR(30) NULL | |
| `role_code` | VARCHAR(100) NULL | |
| `variant_form_id` | UUID | Actual form to use |
| `priority` | INT | |

### 10.3 `metadata_inspector`

Right-side inspector: `inspector_key`, `entity_id`, sections referencing fields/actions.

### 10.4 `metadata_user_preference`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID | IAM ref |
| `list_key` | VARCHAR(150) | |
| `column_order` | JSONB | |
| `hidden_columns` | JSONB | |
| `density` | VARCHAR(20) NULL | |

User prefs cannot reveal FLS-denied fields.

---

## 11. Packages

### 11.1 `metadata_package`

| Column | Type | Notes |
|---|---|---|
| `package_key` | VARCHAR(150) | `india.gst.bp` |
| `version` | VARCHAR(50) | semver |
| `title` | VARCHAR(200) | |
| `checksum` | VARCHAR(64) | |
| `signature` | TEXT NULL | Optional |
| `tenant_id` | UUID NULL | NULL = system pack |
| `installed_at` | TIMESTAMPTZ NULL | |
| `installed_by` | UUID NULL | |
| `status` | VARCHAR(20) | `AVAILABLE`/`INSTALLED`/`DISABLED` |

### 11.2 `metadata_package_item`

| Column | Type | Notes |
|---|---|---|
| `package_id` | UUID | |
| `item_type` | VARCHAR(30) | ENTITY/FIELD/FORM/… |
| `payload` | JSONB | |

---

## 12. Governance

### 12.1 `metadata_approval`

| Column | Type | Notes |
|---|---|---|
| `changeset_id` | UUID | |
| `status` | VARCHAR(20) | |
| `requested_by` | UUID | |
| `reviewed_by` | UUID NULL | |
| `reviewed_at` | TIMESTAMPTZ NULL | |
| `comment` | TEXT NULL | |

### 12.2 `metadata_dependency_edge`

| Column | Type | Notes |
|---|---|---|
| `edge_type` | VARCHAR(40) | |
| `from_type` / `from_key` | | |
| `to_type` / `to_key` | | |
| `is_breaking` | BOOLEAN | |
| `tenant_id` | UUID NULL | |

Maintained on publish and on mutate (async rebuild allowed).

### 12.3 `metadata_catalog_audit`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | |
| `actor_user_id` | UUID NULL | |
| `action` | VARCHAR(40) | |
| `target_type` | VARCHAR(30) | |
| `target_key` | VARCHAR(150) | |
| `before_json` | JSONB NULL | |
| `after_json` | JSONB NULL | |
| `request_id` | UUID NULL | |
| `created_at` | TIMESTAMPTZ | |

### 12.4 Publish tables

Same as v1 (`metadata_publish_version`, `metadata_publish_artifact`) with required `checksum`, `parent_version_number` (rollback lineage), `approval_id` nullable.

---

## 13. Interop descriptors

### 13.1 `metadata_command_descriptor` / `query_descriptor` / `event_descriptor`

| Column | Type | Notes |
|---|---|---|
| `descriptor_key` | VARCHAR(150) | `bp.partner.create` |
| `entity_key` | VARCHAR(150) | |
| `http_method` / `http_path` | | Optional |
| `request_schema` | JSONB | JSON Schema or field list |
| `response_schema` | JSONB | |
| `permission_code` | VARCHAR(100) | |
| `event_type` | VARCHAR(150) | For events |
| `is_system` | BOOLEAN | |

---

## 14. Drift

### 14.1 `metadata_drift_report`

| Column | Type | Notes |
|---|---|---|
| `started_at` / `finished_at` | TIMESTAMPTZ | |
| `status` | VARCHAR(20) | `RUNNING`/`SUCCEEDED`/`FAILED` |
| `scope` | VARCHAR(30) | `SYSTEM`/`TENANT` |
| `tenant_id` | UUID NULL | |
| `finding_count` | INT | |

### 14.2 `metadata_drift_finding`

| Column | Type | Notes |
|---|---|---|
| `report_id` | UUID | |
| `entity_key` | VARCHAR(150) | |
| `field_key` | VARCHAR(100) NULL | |
| `drift_status` | VARCHAR(40) | |
| `expected_json` | JSONB | |
| `actual_json` | JSONB | |
| `severity` | VARCHAR(20) | |

---

## 15. Plumbing

- `metadata_outbox` — standard outbox  
- `metadata_idempotency_key` — standard  
- `metadata_resolve_cache` — optional `(cache_key, payload, etag, expires_at, tenant_id)`

---

## 16. RLS summary

| Table class | Policy |
|---|---|
| System lookups | Readable all; writable platform admin only (app-level) |
| System dictionary with `tenant_id NULL` | Readable with tenant context; manage via permission |
| Tenant overlays/extensions/prefs/EAV | FORCE RLS by `tenant_id` |
| Drift/audit | Tenant-scoped rows RLS; system rows admin |

---

## 17. Seed minimum (advanced)

1. All physical + semantic types + channels + classifications + mask policies  
2. UI controls including accessible variants  
3. Validation types including `CUSTOM_AST`  
4. Modules: `identity`, `org`, `configuration`, `bp`  
5. Domains: `party`, `organization`, `security`, `logistics`  
6. Permission codes (Guide §7)  
7. Sample published version `1` empty or with `org.company` / `bp.partner` starter pack  

---

## 18. ER overview (advanced)

```text
business_domain ──► entity ──► field ──┬── options
                     │                 ├── security
                     │                 ├── defaults/computed (expression)
                     │                 └── search_analyzer
                     ├── relations / indexes / constraints
                     ├── forms/lists/filters/actions (+ variants)
                     ├── measures / dimensions
                     └── command/query/event descriptors

expression ◄── validation / form_field ASTs / computed

package ── items
changeset ── items ── approval ── publish_version ── artifacts
dependency_edge (graph)
drift_report ── findings
catalog_audit
```

---

## 19. Implementation notes

1. Treat publish artifacts as the **runtime source** for effective system layer.  
2. Rebuild `metadata_dependency_edge` transactionally on publish; nightly full reindex OK.  
3. Drift scanner uses DB role with read-only `information_schema` access.  
4. AST schema versioned (`ast_version`); migrations must transform old AST docs.  
5. Split models across `lookups`, `dictionary`, `semantic`, `behavior`, `security`, `ui`, `packs`, `governance`, `interop`, `drift`, `plumbing`.  
6. Never add FK to `org`/`identity` tables.
