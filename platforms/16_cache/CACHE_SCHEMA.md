# JeslotERP Cache Platform — Production Schema (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `cch_namespace` / TTL HTTP persist on AsyncSession. Entry values are ephemeral (L1/L2). Redis L2 is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p16_cache`  
**PostgreSQL schema:** `cache`  
**Companion:** [`CACHE_GUIDE.md`](CACHE_GUIDE.md) · [`CACHE_API.md`](CACHE_API.md)

> Runtime models: `platforms/p16_cache/infrastructure/persistence/models/`.  
> **Note:** Entry **values** primarily live in Redis/L1; Postgres holds control-plane metadata, policies, tag indexes (optional), audits.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `cache` (never `p16`) |
| Tables | `cch_*` |
| Namespace keys | Dot namespaces (`rules.compiled`) |
| Soft delete | Retire namespaces; keep audit |
| Cross-schema | None for values |
| RLS | FORCE on tenant invalidation/warmup jobs |
| Secrets | Backend passwords via configuration `secret_ref` |

---

## 2. Complete table inventory (**58 tables**)

### 2.1 Catalog & key standards (9)

| # | Table | Purpose |
|---|---|---|
| 1 | `cch_namespace` | Cache namespaces |
| 2 | `cch_partition_type` | ORG/TENANT/SESSION/REQUEST/USER |
| 3 | `cch_key_template` | Key templates |
| 4 | `cch_key_param` | Template params |
| 5 | `cch_key_validator` | Validation rules |
| 6 | `cch_tag_def` | Allowed tag prefixes |
| 7 | `cch_owner` | Ownership |
| 8 | `cch_category` | Categories |
| 9 | `cch_feature_binding` | Feature gates |

### 2.2 Policies (8)

| # | Table | Purpose |
|---|---|---|
| 10 | `cch_ttl_policy` | TTL profiles |
| 11 | `cch_size_policy` | Max bytes / items |
| 12 | `cch_eviction_policy` | LRU/LFU hints |
| 13 | `cch_consistency_policy` | Invalidation strength |
| 14 | `cch_sensitivity_policy` | Encryption / forbid |
| 15 | `cch_write_policy` | cache-aside / read-through |
| 16 | `cch_null_cache_policy` | Negative caching |
| 17 | `cch_namespace_policy_bind` | Binds policies → namespace |

### 2.3 Backends & layers (8)

| # | Table | Purpose |
|---|---|---|
| 18 | `cch_backend` | Redis/memory/cluster registry |
| 19 | `cch_backend_secret` | secret_ref |
| 20 | `cch_backend_health` | Health samples |
| 21 | `cch_layer` | L1/L2/L3 definitions |
| 22 | `cch_namespace_backend` | Routing namespace→backend |
| 23 | `cch_cluster_node` | Node inventory |
| 24 | `cch_connection_pool` | Pool settings |
| 25 | `cch_failover_policy` | Failover behavior |

### 2.4 Tags & indexes (6)

| # | Table | Purpose |
|---|---|---|
| 26 | `cch_tag` | Tag keys |
| 27 | `cch_tag_member` | Tag → key_hash index (optional durable) |
| 28 | `cch_key_meta` | Optional meta for critical keys |
| 29 | `cch_key_version` | Logical versions |
| 30 | `cch_tag_stats` | Membership counts |
| 31 | `cch_index_rebuild` | Rebuild jobs |

### 2.5 Invalidation (7)

| # | Table | Purpose |
|---|---|---|
| 32 | `cch_invalidate_request` | Invalidate intents |
| 33 | `cch_invalidate_item` | Keys/tags in request |
| 34 | `cch_invalidate_run` | Execution |
| 35 | `cch_invalidate_bus_cursor` | Bus progress |
| 36 | `cch_purge_request` | Broad purge |
| 37 | `cch_purge_approval` | Prod approvals |
| 38 | `cch_invalidate_audit` | Audit trail |

### 2.6 Single-flight, warmup, quotas (8)

| # | Table | Purpose |
|---|---|---|
| 39 | `cch_lock` | Optional durable lock meta (Redis primary) |
| 40 | `cch_stampede_policy` | Soft TTL / lock TTL |
| 41 | `cch_warmup_plan` | Warmup plans |
| 42 | `cch_warmup_item` | Keys/loaders |
| 43 | `cch_warmup_run` | Runs |
| 44 | `cch_tenant_quota` | Quotas |
| 45 | `cch_tenant_usage` | Usage samples |
| 46 | `cch_quota_breach` | Breach events |

### 2.7 Observability & governance (12)

| # | Table | Purpose |
|---|---|---|
| 47 | `cch_metric_sample` | Hit/miss/latency |
| 48 | `cch_metric_rollup` | Rollups |
| 49 | `cch_slow_key_sample` | Slow keys |
| 50 | `cch_error_sample` | Backend errors |
| 51 | `cch_changeset` | Policy changes |
| 52 | `cch_approval` | Approvals |
| 53 | `cch_package` | Packs |
| 54 | `cch_package_item` | Items |
| 55 | `cch_simulation_run` | Key build dry-run |
| 56 | `cch_sdk_client` | Registered app clients |
| 57 | `cch_alert_rule` | Alert thresholds |
| 58 | `cch_alert_event` | Fired alerts |

**Plumbing:** `cch_outbox`, `cch_idempotency_key`, `cch_catalog_audit`

**Implementation total with plumbing: 61 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `cch_partition` | `ORG`, `TENANT`, `COMPANY`, `USER`, `SESSION`, `REQUEST`, `GLOBAL` |
| `cch_layer_code` | `L1_MEMORY`, `L2_REDIS`, `L3_REMOTE` |
| `cch_write_mode` | `CACHE_ASIDE`, `READ_THROUGH`, `WRITE_THROUGH`, `WRITE_BEHIND` |
| `cch_consistency` | `EVENTUAL`, `STRONG_INVALIDATION` |
| `cch_sensitivity` | `NORMAL`, `SENSITIVE`, `FORBIDDEN` |
| `cch_invalidate_status` | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `PARTIAL` |
| `cch_eviction_hint` | `LRU`, `LFU`, `TTL_ONLY`, `MANUAL` |
| `cch_purge_scope` | `NAMESPACE`, `TENANT`, `BACKEND`, `ALL_ENV_FORBIDDEN` |

---

## 4. Namespace & keys

HTTP namespace/TTL/tag-def persist on `AsyncSession`. Empty list is `[]`. Entry **values** are not SoR — they live in L1 MEMORY or Redis L2 when attached. `buffer_class` is a runtime discipline flag (`GENERIC` / `NONE`); `NONE` refuses `cache_set`. Redis is a port: ping/test-connection are `PROVIDER_PENDING` until a daemon is attached.

### 4.1 `cch_namespace`

| Column | Type | Notes |
|---|---|---|
| `namespace_key` | VARCHAR(100) UNIQUE | `rules.compiled` |
| `name` | VARCHAR(150) | |
| `partition` | VARCHAR(20) | |
| `description` | TEXT NULL | |
| `is_active` | BOOLEAN | |
| `client_side_forbidden` | BOOLEAN DEFAULT true | Browser must not read |
| `owner_platform` | VARCHAR(80) NULL | |

### 4.2 `cch_key_template`

| Column | Type | Notes |
|---|---|---|
| `namespace_id` | UUID | |
| `template_key` | VARCHAR(100) | `entity_effective` |
| `pattern` | VARCHAR(300) | `meta:entity:{tenant_id}:{entity_key}:v{version}` |
| `required_params` | JSONB | |
| `require_tenant_param` | BOOLEAN | |
| `example` | VARCHAR(300) NULL | |

### 4.3 `cch_tag_def`

| Column | Type | Notes |
|---|---|---|
| `prefix` | VARCHAR(50) UNIQUE | `tenant`, `flag`, `rule` |
| `pattern` | VARCHAR(100) | `tenant:{uuid}` |
| `description` | TEXT NULL | |

---

## 5. Policies

### 5.1 `cch_ttl_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | `stable_catalog` |
| `ttl_sec` | INT | Hard TTL |
| `soft_ttl_sec` | INT NULL | Stale-while-revalidate |
| `jitter_sec` | INT DEFAULT 0 | |
| `negative_ttl_sec` | INT NULL | Null-cache |

### 5.2 `cch_sensitivity_policy`

| Column | Type | Notes |
|---|---|---|
| `sensitivity` | VARCHAR(20) | |
| `encrypt_at_rest` | BOOLEAN | |
| `encryption_profile_ref` | VARCHAR(100) NULL | |
| `allow_l1` | BOOLEAN | |
| `forbid_cache` | BOOLEAN | |

### 5.3 `cch_namespace_policy_bind`

One row per namespace linking ttl/size/eviction/consistency/sensitivity/write/null/stampede policies.

---

## 6. Backends

### 6.1 `cch_backend`

| Column | Type | Notes |
|---|---|---|
| `backend_key` | VARCHAR(50) UNIQUE | `redis_primary` |
| `kind` | VARCHAR(30) | `REDIS`, `MEMORY`, `DUMMY` |
| `endpoint` | TEXT NULL | |
| `db_index` | INT NULL | |
| `key_prefix` | VARCHAR(50) | `jeslot:prod:` |
| `secret_ref_key` | VARCHAR(150) NULL | |
| `is_active` | BOOLEAN | |
| `env_key` | VARCHAR(30) | production/staging |

### 6.2 `cch_layer`

| Column | Type | Notes |
|---|---|---|
| `layer_code` | VARCHAR(20) | |
| `backend_id` | UUID NULL | L1 may be null (process memory) |
| `max_entries` | INT NULL | L1 |
| `default_ttl_sec` | INT NULL | |

---

## 7. Tag index & key meta

### 7.1 `cch_tag_member`

| Column | Type | Notes |
|---|---|---|
| `tag` | VARCHAR(200) | `tenant:…` |
| `namespace_id` | UUID | |
| `key_hash` | VARCHAR(64) | sha256 of full key |
| `key_preview` | VARCHAR(200) NULL | Redacted preview |
| `expires_at` | TIMESTAMPTZ NULL | |

Durable tag index optional — Redis sets often primary; PG used for audit/rebuild.

### 7.2 `cch_key_meta`

For critical keys: namespace, key_hash, tags[], size_bytes, set_at, ttl, sensitivity — **not** value.

---

## 8. Invalidation

### 8.1 `cch_invalidate_request`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | |
| `requested_by` | UUID NULL | |
| `reason` | VARCHAR(200) | |
| `status` | VARCHAR(20) | |
| `idempotency_key` | VARCHAR(120) NULL | |
| `created_at` | TIMESTAMPTZ | |

### 8.2 `cch_invalidate_item`

| Column | Type | Notes |
|---|---|---|
| `request_id` | UUID | |
| `item_type` | VARCHAR(20) | KEY / TAG / NAMESPACE / TEMPLATE |
| `value` | VARCHAR(300) | |
| `namespace_key` | VARCHAR(100) NULL | |

### 8.3 `cch_purge_request`

Broad purge; production requires `cch_purge_approval` and never supports ALL without break-glass.

---

## 9. Warmup & quotas

### 9.1 `cch_warmup_plan`

| Column | Type | Notes |
|---|---|---|
| `plan_key` | VARCHAR(100) UNIQUE | |
| `namespace_id` | UUID | |
| `loader_key` | VARCHAR(100) | Allow-listed loader |
| `priority` | INT | |
| `is_active` | BOOLEAN | |

### 9.2 `cch_tenant_quota`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `namespace_id` | UUID NULL | |
| `max_keys` | BIGINT NULL | |
| `max_bytes` | BIGINT NULL | |
| `max_invalidates_per_min` | INT NULL | |

---

## 10. Metrics

### 10.1 `cch_metric_sample`

| Column | Type | Notes |
|---|---|---|
| `namespace_id` | UUID | |
| `layer_code` | VARCHAR(20) | |
| `hits` | BIGINT | |
| `misses` | BIGINT | |
| `errors` | BIGINT | |
| `avg_latency_ms` | NUMERIC | |
| `sampled_at` | TIMESTAMPTZ | |

---

## 11. Governance & packs

- Packages seed namespaces: `metadata.effective`, `i18n.bundle`, `rules.compiled`, `feature.env`, `org.hierarchy`, `notify.prefs`  
- Approvals for production purge & TTL loosening  
- SDK clients registered for audit  

---

## 12. Plumbing

| Table | Purpose |
|---|---|
| `cch_outbox` | Invalidate bus reliability |
| `cch_idempotency_key` | Admin API |
| `cch_catalog_audit` | Policy audit |

---

## 13. RLS summary

| Class | Policy |
|---|---|
| Namespaces/policies system | Read auth; manage permission |
| Invalidate/warmup with tenant | FORCE `tenant_id` |
| Usage/quota | FORCE tenant |
| Backends | Admin only |

---

## 14. Seed minimum

1. Backends: `memory_l1`, `redis_primary` (config-driven)  
2. Layers L1/L2  
3. Namespaces listed in packs  
4. TTL policies: `short_60s`, `medium_300s`, `stable_3600s`, `day_86400s`  
5. Tag defs: tenant, company, user, flag, rule, locale, entity  
6. Stampede policy default  
7. Permissions `cache.*`  

---

## 15. ER overview

```text
namespace ── key_templates / tag_defs
         ── policy binds (ttl/size/sensitivity/…)
         ── backend routing / layers

invalidate_request ── items ── runs / audit
warmup_plan ── runs
tag_member / key_meta
tenant_quota / metric_samples
packages
```

---

## 16. Implementation notes

1. Prefer Redis SET + secondary tag SETs; PG tag_member for rebuild/audit.  
2. L1 must subscribe to invalidate bus — stale L1 is worse than miss.  
3. `FORBIDDEN` sensitivity → SDK throws if set attempted.  
4. Key builder rejects missing tenant when required.  
5. Split models: `catalog`, `policy`, `backend`, `invalidate`, `warmup`, `metrics`, `governance`, `plumbing`.
