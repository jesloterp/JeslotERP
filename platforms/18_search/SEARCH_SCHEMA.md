# JeslotERP Search Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — index/mapping/alias ledger Postgres-first when session is AsyncSession. Documents stay in the engine. Not Production.  
**Package:** `platforms.p18_search`  
**PostgreSQL schema:** `search`  
**Companion:** [`SEARCH_GUIDE.md`](SEARCH_GUIDE.md) · [`SEARCH_API.md`](SEARCH_API.md)

> Runtime models: `platforms/p18_search/infrastructure/persistence/models/`.  
> **Note:** Full-text documents live in the search engine; Postgres holds catalog, pipelines, ACL projections meta, jobs, analytics.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `search` (never `p18`) |
| Tables | `srch_*` |
| Index keys | Dot entity keys (`sales.order`) |
| Soft delete | Tombstone runs; engine delete |
| Cross-schema | `entity_type` + `entity_id` UUIDs only |
| RLS | FORCE on tenant jobs/saved/analytics |
| Engine | Adapter-neutral (OpenSearch/ES/etc.) |

---

## 2. Complete table inventory (**60 tables**)

### 2.1 Catalog & mappings (10)

| # | Table | Purpose |
|---|---|---|
| 1 | `srch_index` | Index catalog |
| 2 | `srch_index_alias` | Aliases (global, blue/green) |
| 3 | `srch_mapping` | Mapping versions |
| 4 | `srch_mapping_field` | Field defs |
| 5 | `srch_field_feature` | searchable/facet/sort/highlight/pii |
| 6 | `srch_mapping_activation` | Active mapping |
| 7 | `srch_entity_binding` | index ↔ metadata entity |
| 8 | `srch_routing_policy` | routing by tenant |
| 9 | `srch_index_settings` | shards/replicas/analyzers ref |
| 10 | `srch_feature_binding` | Feature gates |

### 2.2 Analyzers, synonyms, ranking (8)

| # | Table | Purpose |
|---|---|---|
| 11 | `srch_analyzer` | Analyzer defs |
| 12 | `srch_tokenizer` | Tokenizers |
| 13 | `srch_filter` | Token filters |
| 14 | `srch_synonym_set` | Synonym sets |
| 15 | `srch_synonym_entry` | Entries |
| 16 | `srch_stopword_set` | Stopwords |
| 17 | `srch_rank_profile` | Ranking profiles |
| 18 | `srch_boost_rule` | Field/function boosts |

### 2.3 Pipelines & ingestion (9)

| # | Table | Purpose |
|---|---|---|
| 19 | `srch_pipeline` | Pipeline defs |
| 20 | `srch_pipeline_stage` | Stages |
| 21 | `srch_source_binding` | Event type → pipeline |
| 22 | `srch_ingest_cursor` | Cursors |
| 23 | `srch_ingest_batch` | Batches |
| 24 | `srch_ingest_item` | Item results |
| 25 | `srch_extract_handler` | Allow-listed extractors |
| 26 | `srch_enrichment` | Enrichment hooks |
| 27 | `srch_tombstone` | Deleted entity markers |

### 2.4 ACL projection (5)

| # | Table | Purpose |
|---|---|---|
| 28 | `srch_acl_policy` | How to project ACL |
| 29 | `srch_acl_principal_map` | Optional mapping cache meta |
| 30 | `srch_acl_field` | Field storing principals |
| 31 | `srch_public_flag` | Public-read markers policy |
| 32 | `srch_acl_rebuild` | ACL rebuild jobs |

### 2.5 Query profiles & suggest (7)

| # | Table | Purpose |
|---|---|---|
| 33 | `srch_query_profile` | Profiles (erp_default, mobile) |
| 34 | `srch_query_allow_node` | Allowed DSL nodes |
| 35 | `srch_facet_def` | Facet fields |
| 36 | `srch_sort_def` | Sort fields |
| 37 | `srch_suggest_index` | Completion indexes |
| 38 | `srch_suggest_source` | Source fields |
| 39 | `srch_highlight_policy` | Highlight limits |

### 2.6 Saved search & analytics (6)

| # | Table | Purpose |
|---|---|---|
| 40 | `srch_saved_search` | Saved queries |
| 41 | `srch_saved_share` | Shares |
| 42 | `srch_query_log` | Sampled queries |
| 43 | `srch_zero_result` | Zero-result logs |
| 44 | `srch_click_log` | Click-through |
| 45 | `srch_query_stats` | Aggregates |

### 2.7 Reindex, drift, hybrid (8)

| # | Table | Purpose |
|---|---|---|
| 46 | `srch_reindex_job` | Reindex jobs |
| 47 | `srch_reindex_chunk` | Chunks |
| 48 | `srch_alias_swap` | Blue/green swaps |
| 49 | `srch_drift_scan` | Drift scans |
| 50 | `srch_drift_finding` | Findings |
| 51 | `srch_vector_space` | Optional vector spaces |
| 52 | `srch_embedding_ref` | Embedding provider refs |
| 53 | `srch_hybrid_profile` | Keyword+vector blend |

### 2.8 Engine, governance, packs (7)

| # | Table | Purpose |
|---|---|---|
| 54 | `srch_engine_backend` | Engine cluster registry |
| 55 | `srch_engine_secret` | secret_ref |
| 56 | `srch_changeset` | Mapping changes |
| 57 | `srch_approval` | Approvals |
| 58 | `srch_package` | Packs |
| 59 | `srch_package_item` | Items |
| 60 | `srch_catalog_audit` | Audit |

**Plumbing:** `srch_outbox`, `srch_idempotency_key`

**Implementation total with plumbing: 62 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `srch_field_type` | `TEXT`, `KEYWORD`, `LONG`, `DOUBLE`, `BOOLEAN`, `DATE`, `GEO`, `DENSE_VECTOR` |
| `srch_index_lifecycle` | `DRAFT`, `BUILDING`, `ACTIVE`, `RETIRING`, `RETIRED` |
| `srch_pipeline_stage_kind` | `EXTRACT`, `NORMALIZE`, `ENRICH`, `ACL_PROJECT`, `EMBED`, `INDEX`, `DELETE` |
| `srch_ingest_status` | `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`, `SKIPPED` |
| `srch_reindex_status` | `PENDING`, `RUNNING`, `SWAPPING`, `COMPLETED`, `FAILED`, `CANCELLED` |
| `srch_drift_severity` | `INFO`, `WARNING`, `CRITICAL` |
| `srch_query_node` | `MATCH`, `TERM`, `TERMS`, `RANGE`, `BOOL`, `EXISTS` — deny `SCRIPT`, `RAW` |

---

## 4. Indexes & mappings

### 4.1 `srch_index`

| Column | Type | Notes |
|---|---|---|
| `index_key` | VARCHAR(100) UNIQUE | `sales.order` |
| `display_name` | VARCHAR(150) | |
| `entity_type` | VARCHAR(100) | |
| `lifecycle` | VARCHAR(20) | |
| `engine_index_name` | VARCHAR(150) | Physical name |
| `routing_field` | VARCHAR(50) DEFAULT 'tenant_id' | |
| `is_global_searchable` | BOOLEAN | Include in erp.global |
| `owner_platform` | VARCHAR(80) NULL | |

### 4.2 `srch_mapping_field`

| Column | Type | Notes |
|---|---|---|
| `mapping_id` | UUID | |
| `field_key` | VARCHAR(100) | |
| `field_type` | VARCHAR(20) | |
| `metadata_field_key` | VARCHAR(100) NULL | p05 link |
| `analyzer_key` | VARCHAR(50) NULL | |
| `searchable` | BOOLEAN | |
| `facetable` | BOOLEAN | |
| `sortable` | BOOLEAN | |
| `highlightable` | BOOLEAN | |
| `store` | BOOLEAN | |
| `pii_class` | VARCHAR(20) | |
| `boost` | NUMERIC(6,2) DEFAULT 1 | |

### 4.3 `srch_index_alias`

| Column | Type | Notes |
|---|---|---|
| `alias_key` | VARCHAR(100) | `erp.global`, `sales.order_write` |
| `index_id` | UUID NULL | Target (null if multi) |
| `is_write_alias` | BOOLEAN | |
| `is_read_alias` | BOOLEAN | |

---

## 5. Pipelines

### 5.1 `srch_pipeline_stage`

| Column | Type | Notes |
|---|---|---|
| `pipeline_id` | UUID | |
| `position` | INT | |
| `kind` | VARCHAR(30) | |
| `handler_key` | VARCHAR(100) | Allow-listed |
| `config` | JSONB | |

### 5.2 `srch_source_binding`

| Column | Type | Notes |
|---|---|---|
| `event_type_key` | VARCHAR(200) | p13 type |
| `pipeline_id` | UUID | |
| `index_id` | UUID | |
| `is_active` | BOOLEAN | |

### 5.3 `srch_tombstone`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `entity_type` | VARCHAR(100) | |
| `entity_id` | UUID | |
| `deleted_at` | TIMESTAMPTZ | |
| `source_version` | VARCHAR(64) NULL | |

---

## 6. ACL

### 6.1 `srch_acl_policy`

| Column | Type | Notes |
|---|---|---|
| `index_id` | UUID | |
| `mode` | VARCHAR(30) | `PRINCIPALS_LIST`, `POLICY_KEY`, `TENANT_ONLY` |
| `principal_field` | VARCHAR(50) | `acl_principals` |
| `extractor_key` | VARCHAR(100) | How to build list from source |

---

## 7. Query profiles

### 7.1 `srch_query_profile`

| Column | Type | Notes |
|---|---|---|
| `profile_key` | VARCHAR(50) UNIQUE | `erp_default` |
| `rank_profile_id` | UUID | |
| `default_indexes` | JSONB | |
| `max_size` | INT | |
| `default_highlight` | BOOLEAN | |
| `allow_explain` | BOOLEAN | |

### 7.2 `srch_query_allow_node`

Allow-list of DSL node types for client queries; server builds engine query.

---

## 8. Saved searches & analytics

### 8.1 `srch_saved_search`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID | |
| `name` | VARCHAR(150) | |
| `query_json` | JSONB | |
| `is_shared` | BOOLEAN | |
| `index_keys` | JSONB | |

### 8.2 `srch_query_log`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID NULL | |
| `q` | VARCHAR(500) | |
| `filters_hash` | VARCHAR(64) | |
| `hit_count` | INT | |
| `took_ms` | INT | |
| `created_at` | TIMESTAMPTZ | |

Sampled; PII scrubbing on `q` optional.

---

## 9. Reindex & drift

### 9.1 `srch_reindex_job`

| Column | Type | Notes |
|---|---|---|
| `index_id` | UUID | |
| `target_engine_index` | VARCHAR(150) | New physical |
| `status` | VARCHAR(20) | |
| `mapping_id` | UUID | |
| `total_chunks` | INT | |
| `completed_chunks` | INT | |
| `swap_alias` | BOOLEAN | |

### 9.2 `srch_drift_finding`

| Column | Type | Notes |
|---|---|---|
| `scan_id` | UUID | |
| `entity_id` | UUID | |
| `finding_type` | VARCHAR(40) | MISSING_IN_INDEX, STALE_VERSION, ORPHAN_IN_INDEX |
| `severity` | VARCHAR(20) | |

---

## 10. Hybrid / vector (optional)

### 10.1 `srch_vector_space`

| Column | Type | Notes |
|---|---|---|
| `space_key` | VARCHAR(50) | |
| `dimensions` | INT | |
| `similarity` | VARCHAR(20) | cosine/dot |
| `embedding_ref_key` | VARCHAR(100) | p27/config |

Hybrid profile blends keyword score + vector score with weights.

---

## 11. Engine backend

### 11.1 `srch_engine_backend`

| Column | Type | Notes |
|---|---|---|
| `backend_key` | VARCHAR(50) UNIQUE | |
| `kind` | VARCHAR(30) | `OPENSEARCH`, `ELASTIC`, `MEILI`, `MEMORY` |
| `endpoint` | TEXT | |
| `secret_ref_key` | VARCHAR(150) | |
| `is_active` | BOOLEAN | |

---

## 12. Governance & packs

Packages seed indexes for: `bp.partner`, `sales.order`, `document.document`, `org.company`, `identity.user` (admin-only).  
Global alias `erp.global`.  
Default analyzers `standard_en`, `standard_hi`.

---

## 13. Plumbing

| Table | Purpose |
|---|---|
| `srch_outbox` | Domain events |
| `srch_idempotency_key` | Reindex/admin |

---

## 14. RLS summary

| Class | Policy |
|---|---|
| Index catalog system | Read auth; manage permission |
| Saved searches / query logs | FORCE `tenant_id` |
| Ingest/reindex jobs tenant | FORCE when tenant-scoped |
| Engine secrets | Admin only |

---

## 15. Seed minimum

1. Engine backend placeholder  
2. Query profile `erp_default` + allow nodes  
3. Indexes + mappings for BP and sales order (minimal fields)  
4. Pipelines bound to sample event types  
5. Rank profile default  
6. Permissions `search.*`  
7. Suggest index for partner names / sales order numbers  

---

## 16. ER overview

```text
index ── mappings ── fields / features
    ── aliases / settings / acl_policy
    ── pipeline ── stages / source_bindings
    ── suggest / facets / rank

reindex_job ── chunks / alias_swap
drift_scan ── findings
saved_search / query_log
synonyms / analyzers
engine_backend
packages
```

---

## 17. Implementation notes

1. Physical index names include version suffix; aliases stable.  
2. Upsert doc id = `{tenant_id}:{entity_type}:{entity_id}`.  
3. ACL principals refreshed on role assignment events (rebuild job).  
4. Highlight disabled for `pii_class=RESTRICTED` fields.  
5. Split models: `catalog`, `analysis`, `pipeline`, `acl`, `query`, `jobs`, `analytics`, `governance`, `plumbing`.
