# JeslotERP Feature Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `feat_flag` / `feat_kill_switch` / `feat_tenant_override` / `feat_company_override` are the HTTP ledger. Empty list is `[]`. Not Production.  
**Package:** `platforms.p12_feature`  
**PostgreSQL schema:** `feature`  
**Companion:** [`FEATURE_GUIDE.md`](FEATURE_GUIDE.md) · [`FEATURE_API.md`](FEATURE_API.md)

> Runtime models: `platforms/p12_feature/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `feature` (never `p12`) |
| Tables | `feat_*` |
| Flag keys | Dot namespaces (`sales.order.bulk_import.v2`) |
| Soft delete | Archive flags; keep eval history |
| Cross-schema | UUID in context only |
| RLS | FORCE on tenant overrides / tenant segments |
| Variations | Immutable ids within flag; add new rather than rewrite meaning |

---

## 2. Complete table inventory (**60 tables**)

### 2.1 Catalog & taxonomy (8)

| # | Table | Purpose |
|---|---|---|
| 1 | `feat_flag` | Flag catalog |
| 2 | `feat_flag_alias` | Deprecated key → canonical |
| 3 | `feat_tag` | Tags |
| 4 | `feat_flag_tag` | M2M |
| 5 | `feat_owner` | Ownership |
| 6 | `feat_category` | Categories/modules |
| 7 | `feat_flag_category` | M2M |
| 8 | `feat_debt` | Temporary flag debt tracker |

### 2.2 Variations & types (5)

| # | Table | Purpose |
|---|---|---|
| 9 | `feat_variation` | Variation definitions |
| 10 | `feat_flag_default` | Default on/off/fallthrough indexes |
| 11 | `feat_value_schema` | JSON schema for JSON flags |
| 12 | `feat_variation_label` | i18n label keys |
| 13 | `feat_flag_type_constraint` | Type constraints |

### 2.3 Environments & delivery (8)

| # | Table | Purpose |
|---|---|---|
| 14 | `feat_environment` | development/staging/production |
| 15 | `feat_env_flag_config` | Per-env flag config header |
| 16 | `feat_targeting_rule` | Ordered rules |
| 17 | `feat_rule_clause` | Rule clauses |
| 18 | `feat_rule_serve` | Variation served by rule |
| 19 | `feat_rollout` | Percentage rollout config |
| 20 | `feat_rollout_weight` | Variation weights |
| 21 | `feat_fallthrough` | Fallthrough serve/rollout |

### 2.4 Segments (6)

| # | Table | Purpose |
|---|---|---|
| 22 | `feat_segment` | Segment header |
| 23 | `feat_segment_rule` | Segment match rules |
| 24 | `feat_segment_clause` | Clauses |
| 25 | `feat_segment_member` | Explicit include/exclude ids |
| 26 | `feat_segment_scope` | System vs tenant segment |
| 27 | `feat_rule_segment_ref` | Rule → segment |

### 2.5 Prerequisites, kill, overrides (7)

| # | Table | Purpose |
|---|---|---|
| 28 | `feat_prerequisite` | Flag depends on flag+variation |
| 29 | `feat_kill_switch` | Engaged kill records |
| 30 | `feat_tenant_override` | Tenant FORCE_* |
| 31 | `feat_company_override` | Company FORCE_* |
| 32 | `feat_user_override` | Optional user force (support) |
| 33 | `feat_override_reason` | Reason codes |
| 34 | `feat_break_glass` | Break-glass allow during kill |

### 2.6 Bucketing & evaluation aids (5)

| # | Table | Purpose |
|---|---|---|
| 35 | `feat_bucket_salt` | Per-flag/env salt |
| 36 | `feat_sticky_assignment` | Optional durable sticky rows |
| 37 | `feat_eval_cache_key` | Cache fingerprint helpers |
| 38 | `feat_bootstrap_snapshot` | Published bootstrap blobs |
| 39 | `feat_reason_code` | Reason catalog |

### 2.7 Experiments & exposure (6)

| # | Table | Purpose |
|---|---|---|
| 40 | `feat_experiment` | Experiment header |
| 41 | `feat_experiment_flag` | Bound flags |
| 42 | `feat_experiment_metric` | Metric keys (external) |
| 43 | `feat_exposure` | Exposure log |
| 44 | `feat_exposure_dedupe` | Subject dedupe |
| 45 | `feat_experiment_guardrail` | Limits |

### 2.8 Scheduling & promote (5)

| # | Table | Purpose |
|---|---|---|
| 46 | `feat_schedule` | Scheduled changes |
| 47 | `feat_schedule_action` | Actions to apply |
| 48 | `feat_schedule_run` | Execution history |
| 49 | `feat_env_promote` | Promote rules across envs |
| 50 | `feat_promote_diff` | Diff snapshots |

### 2.9 Governance, packs, audit (10)

| # | Table | Purpose |
|---|---|---|
| 51 | `feat_changeset` | Change batches |
| 52 | `feat_approval` | Approvals |
| 53 | `feat_package` | Feature packs |
| 54 | `feat_package_item` | Pack items |
| 55 | `feat_change_audit` | Flag config audit |
| 56 | `feat_eval_audit` | Sampled eval audit |
| 57 | `feat_usage_stats` | Eval counts |
| 58 | `feat_sdk_key` | Server/client SDK identifiers (not secrets raw) |
| 59 | `feat_sdk_key_secret_ref` | secret_ref to configuration |
| 60 | `feat_webhook` | Change webhooks |

**Plumbing:** `feat_outbox`, `feat_idempotency_key`, `feat_catalog_audit`

**Implementation total with plumbing: 63 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `feat_flag_type` | `BOOLEAN`, `STRING`, `NUMBER`, `JSON`, `VARIANT` |
| `feat_flag_status` | `ACTIVE`, `INACTIVE`, `ARCHIVED` |
| `feat_clause_op` | `EQ`, `NEQ`, `IN`, `NOT_IN`, `CONTAINS`, `STARTS_WITH`, `ENDS_WITH`, `GT`, `GTE`, `LT`, `LTE`, `SEMVER_EQ`, `SEMVER_GTE`, `EXISTS`, `NOT_EXISTS`, `SEGMENT_MATCH` |
| `feat_override_mode` | `FORCE_ON`, `FORCE_OFF`, `FORCE_VARIATION` |
| `feat_serve_kind` | `VARIATION`, `ROLLOUT` |
| `feat_schedule_status` | `PENDING`, `APPLIED`, `CANCELLED`, `FAILED` |
| `feat_experiment_status` | `DRAFT`, `RUNNING`, `PAUSED`, `STOPPED` |
| `feat_reason` | `KILL_SWITCH`, `ENV_OFF`, `PREREQ_FAILED`, `OVERRIDE`, `RULE_MATCH`, `PERCENT_ROLLOUT`, `FALLTHROUGH`, `OFF`, `ERROR_DEFAULT` |
| `feat_segment_kind` | `RULE_BASED`, `LIST_BASED`, `MIXED` |

---

## 4. Flag catalog (detail)

### 4.1 `feat_flag`

| Column | Type | Notes |
|---|---|---|
| `flag_key` | VARCHAR(150) UNIQUE | |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `flag_type` | VARCHAR(20) | |
| `status` | VARCHAR(20) | |
| `is_temporary` | BOOLEAN | Debt candidate |
| `owner_team` | VARCHAR(100) NULL | |
| `client_side_available` | BOOLEAN | Bootstrap eligible |
| `default_bucket_attr` | VARCHAR(50) | `user_id` / `tenant_id` |
| `created_at` | TIMESTAMPTZ | |

### 4.2 `feat_variation`

| Column | Type | Notes |
|---|---|---|
| `flag_id` | UUID | |
| `variation_key` | VARCHAR(80) | `on`, `off`, `v2_ui` |
| `value_json` | JSONB | Typed value |
| `description` | VARCHAR(200) NULL | |
| `position` | INT | |
| `is_off_variation` | BOOLEAN | |

**Unique:** `(flag_id, variation_key)`.

---

## 5. Environment delivery

### 5.1 `feat_environment`

| Column | Type | Notes |
|---|---|---|
| `env_key` | VARCHAR(30) UNIQUE | `production` |
| `name` | VARCHAR(80) | |
| `is_production` | BOOLEAN | Stricter approvals |

### 5.2 `feat_env_flag_config`

| Column | Type | Notes |
|---|---|---|
| `flag_id` | UUID | |
| `environment_id` | UUID | |
| `is_enabled` | BOOLEAN | Master env toggle |
| `version` | INT | Optimistic concurrency |
| `checksum` | VARCHAR(64) | Rules hash |
| `updated_at` | TIMESTAMPTZ | |

**Unique:** `(flag_id, environment_id)`.

### 5.3 `feat_targeting_rule`

| Column | Type | Notes |
|---|---|---|
| `env_flag_config_id` | UUID | |
| `position` | INT | FIRST match |
| `description` | VARCHAR(200) NULL | |
| `segment_id` | UUID NULL | Optional |
| `serve_kind` | VARCHAR(20) | VARIATION/ROLLOUT |
| `variation_id` | UUID NULL | |
| `rollout_id` | UUID NULL | |
| `is_enabled` | BOOLEAN | |

### 5.4 `feat_rule_clause`

| Column | Type | Notes |
|---|---|---|
| `rule_id` | UUID | |
| `position` | INT | AND within rule |
| `attribute` | VARCHAR(80) | `tenant_id`, `roles`, `plan_code`, `custom.region` |
| `op` | VARCHAR(30) | |
| `value_json` | JSONB | |

### 5.5 `feat_rollout` / weights

| Column | Type | Notes |
|---|---|---|
| `bucket_by` | VARCHAR(50) | Context attr |
| `salt` | VARCHAR(64) | |
| weights | variation_id + weight_bps (sum 10000) | |

---

## 6. Segments

### 6.1 `feat_segment`

| Column | Type | Notes |
|---|---|---|
| `segment_key` | VARCHAR(100) | |
| `name` | VARCHAR(150) | |
| `kind` | VARCHAR(20) | |
| `tenant_id` | UUID NULL | Null = system |
| `description` | TEXT NULL | |

### 6.2 `feat_segment_member`

Include/exclude lists for tenant_id/user_id/company_id with `member_type` + `member_id`.

---

## 7. Prerequisites, kill, overrides

### 7.1 `feat_prerequisite`

| Column | Type | Notes |
|---|---|---|
| `flag_id` | UUID | Dependent |
| `requires_flag_id` | UUID | |
| `requires_variation_id` | UUID NULL | Null = any non-off |
| `environment_id` | UUID NULL | Null = all envs |

### 7.2 `feat_kill_switch`

| Column | Type | Notes |
|---|---|---|
| `flag_id` | UUID | |
| `environment_id` | UUID NULL | Null = all |
| `engaged_by` | UUID | |
| `engaged_at` | TIMESTAMPTZ | |
| `reason` | TEXT | |
| `cleared_at` | TIMESTAMPTZ NULL | |
| `is_active` | BOOLEAN | |

### 7.3 `feat_tenant_override`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | RLS |
| `flag_id` | UUID | |
| `environment_id` | UUID | |
| `mode` | VARCHAR(20) | |
| `variation_id` | UUID NULL | |
| `reason_code` | VARCHAR(50) | |
| `expires_at` | TIMESTAMPTZ NULL | |
| `created_by` | UUID | |

**Unique live:** `(tenant_id, flag_id, environment_id)`.

---

## 8. Experiments & exposure

### 8.1 `feat_experiment`

| Column | Type | Notes |
|---|---|---|
| `experiment_key` | VARCHAR(100) UNIQUE | |
| `hypothesis` | TEXT NULL | |
| `status` | VARCHAR(20) | |
| `started_at` / `ended_at` | TIMESTAMPTZ | |

### 8.2 `feat_exposure`

| Column | Type | Notes |
|---|---|---|
| `experiment_id` | UUID NULL | |
| `flag_id` | UUID | |
| `environment_id` | UUID | |
| `tenant_id` | UUID | |
| `subject_key` | VARCHAR(120) | bucket key |
| `variation_id` | UUID | |
| `exposed_at` | TIMESTAMPTZ | |

Dedupe via `feat_exposure_dedupe` unique `(flag_id, env, subject_key)`.

---

## 9. Schedules & promote

### 9.1 `feat_schedule`

| Column | Type | Notes |
|---|---|---|
| `schedule_key` | VARCHAR(100) | |
| `flag_id` | UUID | |
| `environment_id` | UUID | |
| `execute_at` | TIMESTAMPTZ | |
| `status` | VARCHAR(20) | |
| `payload` | JSONB | Patch to apply |
| `created_by` | UUID | |

### 9.2 `feat_env_promote`

Copies/diffs targeting from staging → production with approval.

---

## 10. Bootstrap & SDK

### 10.1 `feat_bootstrap_snapshot`

| Column | Type | Notes |
|---|---|---|
| `environment_id` | UUID | |
| `scope_hash` | VARCHAR(64) | tenant/company/user fingerprint |
| `etag` | VARCHAR(64) | |
| `payload` | JSONB | Evaluated client-side flags |
| `expires_at` | TIMESTAMPTZ | |

### 10.2 SDK keys

Public SDK identifiers + `secret_ref_key` for server SDKs stored via configuration secrets.

---

## 11. Governance & packs

- Changesets/approvals especially for production env  
- Packages: `transport.features.v1`, `finance.features.v1`  
- Webhooks on kill/override/rule change  

---

## 12. Plumbing

| Table | Purpose |
|---|---|
| `feat_outbox` | Events |
| `feat_idempotency_key` | Kill/override/schedule |
| `feat_catalog_audit` | Catalog before/after |

---

## 13. RLS summary

| Class | Policy |
|---|---|
| Flags/env configs system | Read auth; manage permission |
| Tenant/company/user overrides | FORCE `tenant_id` |
| Tenant segments / members | FORCE `tenant_id` |
| Exposures | FORCE `tenant_id` |

---

## 14. Seed minimum

1. Environments: `development`, `staging`, `production`  
2. Reason codes catalog  
3. Sample flags: `sales.order.bulk_import.v2`, `ui.shell.dense_mode`, `finance.credit.workflow.v2`  
4. Variations on/off (+ multivariate sample)  
5. Segment `beta_tenants` (list-based empty)  
6. Permissions `feature.*`  
7. Bootstrap eligible flags marked  

---

## 15. ER overview

```text
flag ── variations
  │
  ├── env_flag_config ── rules ── clauses / segment_ref
  │                   ── rollout / fallthrough
  │                   ── prerequisites
  ├── kill_switch
  ├── tenant/company/user overrides
  ├── schedules / promote
  └── experiments ── exposures

segment ── rules / members
packages / changesets / bootstrap_snapshot
```

---

## 16. Implementation notes

1. Evaluator must be side-effect free except optional exposure write (async).  
2. Weight_bps must sum to 10000.  
3. Archive flag → evaluate returns off variation.  
4. Production rule edits may require changeset approval when policy enabled.  
5. Split models: `catalog`, `delivery`, `segment`, `override`, `experiment`, `schedule`, `bootstrap`, `plumbing`.
