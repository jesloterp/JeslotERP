# Search Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p18_search/tests -q` → **29 passed**  
**Status:** **SoR-Live** (not Production; no live OpenSearch soak)

| Requirement ID | Source | Requirement | Component | Status | Tests | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| SRCH-G-01 | GUIDE §1 | Derived indexes, not ILIKE OLTP | MEMORY engine + catalog | Implemented | module + query | PASS |
| SRCH-G-02 | GUIDE §4 | Tenant + ACL trim on every query | `catalog_store._visible` | Implemented | `test_tenant_isolation_and_acl_deny` | PASS |
| SRCH-G-03 | GUIDE §3 | Idempotent upsert source_version | `upsert` | Implemented | `test_idempotent_upsert_and_drift_missing` | PASS |
| SRCH-G-04 | GUIDE §3 | Blue/green reindex alias swap | `start_reindex` / chunk run | Implemented | `test_reindex_swap_and_internal_index_delete` | PASS |
| SRCH-G-05 | GUIDE §3 | Allow-listed DSL; deny SCRIPT/RAW | query + DENIED_DSL_NODES | Implemented | `test_dsl_script_denied_and_facet_not_allowed` | PASS |
| SRCH-G-06 | GUIDE §3 | Facets only declared fields | facet defs | Implemented | facet 422 test | PASS |
| SRCH-G-07 | GUIDE §5 | en/hi analyzers | seed `standard_en/hi` | Implemented | `test_analyzers_en_hi_seeded` | PASS |
| SRCH-G-08 | GUIDE §11 | Drift finds missing docs | drift scan + repair | Implemented | drift tests | PASS |
| SRCH-G-09 | GUIDE §7 | Permissions search.* | catalog + HTTP gates | Implemented | 403 denied | PASS |
| SRCH-S-01 | SCHEMA §2 | 60 + outbox + idempotency = 62 | ORM | Implemented | `test_srch_module_tables_count_62` | PASS |
| SRCH-S-02 | SCHEMA §15 | Seed engine, profile, BP/bilty, packs | `seed_defaults` | Implemented | catalog list | PASS |
| SRCH-A-01 | API §6 | query/global/suggest/mget/explain | query router | Implemented | API contracts | PASS |
| SRCH-A-02 | API §7 | Saved searches CRUD/execute/share | `/saved` | Implemented | `test_saved_crud_execute_and_missing` | PASS |
| SRCH-A-03 | API §8–9 | Indexes/mappings/aliases/analyzers/synonyms | catalog router | Implemented | catalog tests | PASS |
| SRCH-A-04 | API §10–11 | Internal index/delete; reindex; drift | internal + ops | Implemented | reindex + drift | PASS |
| SRCH-A-05 | API §12 | Hybrid unavailable by default | `/hybrid` | Implemented | hybrid 422 | PASS |
| SRCH-A-06 | API §13–15 | Analytics, packages, backends, health | ops | Implemented | analytics/health | PASS |
| SRCH-SOR-07 | TASK-SOR-007 | OpenSearch live path; MEMORY test double | `OpenSearchEngine` + `try_attach_opensearch` | Implemented | `test_store_health_stays_memory_under_pytest` | PASS |
| SRCH-SOR-07b | TASK-SOR-007 | Empty catalog is `[]`; RLS on HTTP | catalog_repository + `require_search_access` | Implemented | `test_durable_sor` | PASS |
| SRCH-MOD | registry | ModulePlugin deps p05+p08+p09 | SearchModule + main/env | Implemented | load order | PASS |
