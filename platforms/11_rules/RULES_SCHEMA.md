# JeslotERP Rules Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — eval log + compiled artifact HTTP persist on AsyncSession. Alembic `e9f0a1b2c3d4` / `f0a1b2c3d4e5`. Not Production.  
**Package:** `platforms.p11_rules`  
**PostgreSQL schema:** `rules`  
**Companion:** [`RULES_GUIDE.md`](RULES_GUIDE.md) · [`RULES_API.md`](RULES_API.md)

> Runtime models: `platforms/p11_rules/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `rules` (never `p11`) |
| Tables | `rule_*` |
| Soft delete | Supersede versions; overlays disable rows |
| Cross-schema | UUID refs only; facts are JSON |
| RLS | FORCE on tenant overlays & eval logs |
| Immutability | ACTIVE compiled artifacts read-only |
| AST | JSON AST + optional source text |

---

## 2. Complete table inventory (**62 tables**)

### 2.1 Catalog (8)

| # | Table | Purpose |
|---|---|---|
| 1 | `rule_artifact` | Catalog entry (rule_key, kind) |
| 2 | `rule_category` | Categories |
| 3 | `rule_artifact_category` | M2M |
| 4 | `rule_tag` | Tags |
| 5 | `rule_artifact_tag` | M2M |
| 6 | `rule_owner` | Ownership |
| 7 | `rule_feature_binding` | Feature gates |
| 8 | `rule_sensitivity` | Audit/explain requirements |

### 2.2 Definitions & compile (7)

| # | Table | Purpose |
|---|---|---|
| 9 | `rule_definition` | Version header |
| 10 | `rule_definition_source` | Human source (JSON/DMN-ish) |
| 11 | `rule_compiled_artifact` | Immutable compile output |
| 12 | `rule_checksum` | Content hashes |
| 13 | `rule_activation` | Which version ACTIVE per scope |
| 14 | `rule_dependency` | Artifact depends on artifact |
| 15 | `rule_compat` | Compatible metadata/field refs |

### 2.3 Decision tables (9)

| # | Table | Purpose |
|---|---|---|
| 16 | `rule_decision_table` | Table header on definition |
| 17 | `rule_dt_input` | Input clauses |
| 18 | `rule_dt_output` | Output clauses |
| 19 | `rule_dt_row` | Rows |
| 20 | `rule_dt_cell` | Cell entries |
| 21 | `rule_dt_hit_policy` | Policy + collect aggregator |
| 22 | `rule_dt_annotation` | Notes per row |
| 23 | `rule_dt_priority` | Priority values |
| 24 | `rule_dt_effective_date` | Row validity window |

### 2.4 Expressions & functions (6)

| # | Table | Purpose |
|---|---|---|
| 25 | `rule_expression` | Expression artifact body |
| 26 | `rule_ast_node_allow` | Allow-listed node types |
| 27 | `rule_function` | Named reusable functions |
| 28 | `rule_function_param` | Function params |
| 29 | `rule_function_body` | Function AST |
| 30 | `rule_literal_library` | Named constants |

### 2.5 Rule sets, validation, assignment (7)

| # | Table | Purpose |
|---|---|---|
| 31 | `rule_set` | Ordered set header |
| 32 | `rule_set_member` | Members |
| 33 | `rule_validation` | Validation rule meta |
| 34 | `rule_validation_message` | message_key / severity |
| 35 | `rule_assignment_table` | Assignment matrix header |
| 36 | `rule_assignment_row` | Rows → queue/role/user hint |
| 37 | `rule_scorecard` | Optional weighted card |

### 2.6 Facts, context, actions (7)

| # | Table | Purpose |
|---|---|---|
| 38 | `rule_fact_schema` | Declared facts |
| 39 | `rule_fact_field` | Fact fields/types |
| 40 | `rule_context_provider` | Provider registry |
| 41 | `rule_context_binding` | Artifact → providers |
| 42 | `rule_action_def` | Action catalog |
| 43 | `rule_action_param` | Action params |
| 44 | `rule_action_binding` | Row/expr → suggested action |

### 2.7 Overlays (5)

| # | Table | Purpose |
|---|---|---|
| 45 | `rule_overlay` | Tenant/company overlay header |
| 46 | `rule_overlay_item` | ADD/DISABLE/REPLACE ops |
| 47 | `rule_overlay_row` | Overlay table rows |
| 48 | `rule_overlay_cell` | Overlay cells |
| 49 | `rule_overlay_activation` | Active overlay versions |

### 2.8 Simulation, eval, governance (8)

| # | Table | Purpose |
|---|---|---|
| 50 | `rule_test_suite` | Simulation suite |
| 51 | `rule_test_case` | Cases |
| 52 | `rule_test_run` | Run header |
| 53 | `rule_test_result` | Per-case results |
| 54 | `rule_eval_log` | Optional runtime eval audit |
| 55 | `rule_explain_snapshot` | Stored explain for sensitive |
| 56 | `rule_changeset` | Change batches |
| 57 | `rule_approval` | Approvals |

### 2.9 Packs & metrics (5)

| # | Table | Purpose |
|---|---|---|
| 58 | `rule_package` | Packs |
| 59 | `rule_package_item` | Items |
| 60 | `rule_usage_stats` | Invoke counts |
| 61 | `rule_perf_sample` | Latency samples |
| 62 | `rule_budget` | Complexity budgets |

**Plumbing:** `rule_outbox`, `rule_idempotency_key`, `rule_catalog_audit`

**Implementation total with plumbing: 65 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `rule_artifact_kind` | `DECISION_TABLE`, `EXPRESSION`, `RULE_SET`, `DECISION_CHAIN`, `VALIDATION_SET`, `ASSIGNMENT_TABLE`, `SCORECARD`, `FUNCTION` |
| `rule_def_lifecycle` | `DRAFT`, `PUBLISHED`, `ACTIVE`, `RETIRED` |
| `rule_hit_policy` | `FIRST`, `UNIQUE`, `ANY`, `PRIORITY`, `COLLECT` |
| `rule_collect_agg` | `LIST`, `SUM`, `MIN`, `MAX`, `COUNT`, `ALL_TRUE`, `ANY_TRUE` |
| `rule_cell_op` | `ANY`, `EQ`, `NEQ`, `GT`, `GTE`, `LT`, `LTE`, `IN`, `NOT_IN`, `BETWEEN`, `IS_NULL`, `EXPR` |
| `rule_severity` | `INFO`, `WARNING`, `ERROR`, `BLOCKER` |
| `rule_overlay_op` | `ADD_ROW`, `DISABLE_ROW`, `REPLACE_ROW`, `REPLACE_EXPR`, `DISABLE_ARTIFACT` |
| `rule_eval_status` | `OK`, `NO_HIT`, `MULTI_HIT_ERROR`, `TIMEOUT`, `TYPE_ERROR`, `AST_DENIED`, `FACT_INVALID` |
| `rule_layer` | `SYSTEM`, `PACKAGE`, `TENANT`, `COMPANY` |

---

## 4. Catalog & definitions

### 4.1 `rule_artifact`

| Column | Type | Notes |
|---|---|---|
| `rule_key` | VARCHAR(150) UNIQUE | `finance.credit.requires_approval` |
| `kind` | VARCHAR(30) | |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `label_key` | VARCHAR(200) NULL | i18n |
| `sensitivity` | VARCHAR(20) | NORMAL/SENSITIVE |
| `require_explain` | BOOLEAN | |
| `strict_facts` | BOOLEAN | |
| `is_active` | BOOLEAN | |

### 4.2 `rule_definition`

| Column | Type | Notes |
|---|---|---|
| `artifact_id` | UUID | |
| `version_number` | INT | |
| `lifecycle` | VARCHAR(20) | |
| `checksum` | VARCHAR(64) | |
| `compiled_artifact_id` | UUID NULL | |
| `published_at` / `activated_at` | TIMESTAMPTZ | |
| `max_eval_ms` | INT NULL | |
| `notes` | TEXT NULL | |

**Unique:** `(artifact_id, version_number)`.

### 4.3 `rule_compiled_artifact`

| Column | Type | Notes |
|---|---|---|
| `definition_id` | UUID | |
| `compiler_version` | VARCHAR(20) | |
| `payload` | JSONB | Optimized structure |
| `checksum` | VARCHAR(64) | |
| `created_at` | TIMESTAMPTZ | |

Immutable once written.

---

## 5. Decision tables

### 5.1 `rule_decision_table`

| Column | Type | Notes |
|---|---|---|
| `definition_id` | UUID | |
| `hit_policy` | VARCHAR(20) | |
| `collect_agg` | VARCHAR(20) NULL | |
| `prefer_order` | VARCHAR(20) | ROW_ORDER / PRIORITY |

### 5.2 `rule_dt_input` / `rule_dt_output`

| Column | Type | Notes |
|---|---|---|
| `table_id` | UUID | |
| `position` | INT | |
| `name` | VARCHAR(80) | |
| `fact_path` | VARCHAR(150) | `amount`, `company.type` |
| `data_type` | VARCHAR(20) | NUMBER/STRING/BOOL/DATE/… |
| `label_key` | VARCHAR(200) NULL | |

### 5.3 `rule_dt_row`

| Column | Type | Notes |
|---|---|---|
| `table_id` | UUID | |
| `position` | INT | |
| `priority` | INT NULL | |
| `is_enabled` | BOOLEAN | |
| `valid_from` / `valid_to` | TIMESTAMPTZ NULL | |
| `annotation` | TEXT NULL | |

### 5.4 `rule_dt_cell`

| Column | Type | Notes |
|---|---|---|
| `row_id` | UUID | |
| `clause_id` | UUID | input or output clause |
| `side` | VARCHAR(10) | INPUT/OUTPUT |
| `op` | VARCHAR(20) | |
| `value_json` | JSONB NULL | |
| `expr_ast` | JSONB NULL | When op=EXPR |

---

## 6. Expressions & functions

### 6.1 `rule_expression`

| Column | Type | Notes |
|---|---|---|
| `definition_id` | UUID | |
| `return_type` | VARCHAR(20) | |
| `source_text` | TEXT NULL | Display |
| `ast` | JSONB NOT NULL | |

### 6.2 `rule_function`

Reusable named functions with params; body AST; callable from expressions if allow-listed.

---

## 7. Rule sets & validation

### 7.1 `rule_set_member`

| Column | Type | Notes |
|---|---|---|
| `set_definition_id` | UUID | |
| `member_artifact_id` | UUID | |
| `position` | INT | |
| `stop_on_blocker` | BOOLEAN | For validation sets |
| `enabled` | BOOLEAN | |

### 7.2 `rule_validation_message`

| Column | Type | Notes |
|---|---|---|
| `validation_id` | UUID | |
| `severity` | VARCHAR(20) | |
| `message_key` | VARCHAR(200) | i18n |
| `message_params_paths` | JSONB NULL | Fact paths for ICU |

---

## 8. Facts & context

### 8.1 `rule_fact_field`

| Column | Type | Notes |
|---|---|---|
| `schema_id` | UUID | |
| `path` | VARCHAR(150) | |
| `data_type` | VARCHAR(20) | |
| `required` | BOOLEAN | |
| `enum_values` | JSONB NULL | |

### 8.2 `rule_context_provider`

| Column | Type | Notes |
|---|---|---|
| `provider_key` | VARCHAR(50) UNIQUE | `org.company` |
| `timeout_ms` | INT | |
| `allowed_output_paths` | JSONB | |
| `is_active` | BOOLEAN | |

---

## 9. Overlays

### 9.1 `rule_overlay`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | |
| `company_id` | UUID NULL | |
| `base_artifact_id` | UUID | |
| `base_definition_id` | UUID | Pinned base |
| `version_number` | INT | |
| `lifecycle` | VARCHAR(20) | |
| `checksum` | VARCHAR(64) | |

### 9.2 `rule_overlay_item`

| Column | Type | Notes |
|---|---|---|
| `overlay_id` | UUID | |
| `op` | VARCHAR(30) | |
| `target_row_ref` | VARCHAR(80) NULL | Stable row key |
| `payload` | JSONB | New row/expr |

---

## 10. Simulation & eval log

### 10.1 `rule_test_case`

| Column | Type | Notes |
|---|---|---|
| `suite_id` | UUID | |
| `name` | VARCHAR(150) | |
| `facts` | JSONB | |
| `expected_result` | JSONB | |
| `expected_row_key` | VARCHAR(80) NULL | |
| `expect_status` | VARCHAR(20) | OK/NO_HIT/… |

### 10.2 `rule_eval_log`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `artifact_id` | UUID | |
| `definition_id` | UUID | |
| `evaluation_id` | UUID | |
| `facts_hash` | VARCHAR(64) | Not full facts if sensitive |
| `result_hash` | VARCHAR(64) | |
| `status` | VARCHAR(20) | |
| `duration_ms` | INT | |
| `caller` | VARCHAR(50) | process/domain/api |
| `created_at` | TIMESTAMPTZ | |

Full facts stored only when policy `store_facts=true` and not sensitive — else explain snapshot separate with ACL.

---

## 11. Actions

### 11.1 `rule_action_def`

| Column | Type | Notes |
|---|---|---|
| `action_key` | VARCHAR(100) UNIQUE | `domain.validation.reject` |
| `description` | TEXT | |
| `is_side_effecting` | BOOLEAN | |
| `allowed_callers` | JSONB | `["domain","process"]` |

### 11.2 `rule_action_binding`

Binds table row or expression outcome to suggested action + param mapping from facts/outputs.

---

## 12. Governance & packs

- Changesets/approvals before activate  
- Packages: `finance.credit.matrix@1.0.0`, `sales.order.validation@1.0.0`, `process.gates.common@1.0.0`  
- Budgets: max rows, max AST depth, max eval ms defaults  

---

## 13. Plumbing

| Table | Purpose |
|---|---|
| `rule_outbox` | Events |
| `rule_idempotency_key` | Publish/install |
| `rule_catalog_audit` | Catalog before/after |

---

## 14. RLS summary

| Class | Policy |
|---|---|
| System artifacts | Read auth; manage permission |
| Overlays | FORCE `tenant_id` |
| Eval logs / explain snapshots | FORCE tenant + sensitivity ACL |
| Test suites | Tenant or system |

---

## 15. Seed minimum

1. AST allow-list rows  
2. Artifacts: `process.gate.amount_gt_100k`, `finance.credit.requires_approval`, `sales.order.validation`  
3. Sample decision tables ACTIVE  
4. Fact schemas for each  
5. Actions: `domain.validation.reject`, `process.edge.select` (logical)  
6. Context providers: `org.company`, `feature.flags`  
7. Budgets defaults  
8. Permissions `rules.*`  

---

## 16. ER overview

```text
artifact ── definitions ── compiled_artifact
                │
                ├─ decision_table ── inputs/outputs/rows/cells
                ├─ expression / functions
                ├─ rule_set members / validations / assignment
                └─ fact_schema / context_bindings / action_bindings

overlay (tenant/company) ── items/rows
test_suite ── cases ── runs/results
packages / changesets / eval_log
```

---

## 17. Implementation notes

1. Compile validates types across inputs/outputs/AST.  
2. UNIQUE hit policy with 0 or >1 matches → `MULTI_HIT_ERROR` / `NO_HIT`.  
3. Overlay merge happens before compile-cache key resolution.  
4. Never execute action_defs inside pure `/evaluate` unless explicitly `apply=true` + permission.  
5. Split models: `catalog`, `table`, `expr`, `set`, `overlay`, `runtime`, `sim`, `governance`, `plumbing`.
