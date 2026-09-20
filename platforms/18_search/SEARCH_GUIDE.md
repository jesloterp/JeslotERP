# JeslotERP Search Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — OpenSearch attaches when ping succeeds; MEMORY is the TestClient/pytest double; catalog empty list is `[]`. Not Production.  
**Package:** `platforms.p18_search`  
**PostgreSQL schema:** `search`  
**Depends on:** `p05_metadata`, `p08_file_media`, `p09_document`  
**Integrates with:** `p01_identity` (ACL), `p02_organization`, `p06_localization` (analyzers/locales), `p11_rules` (optional boosts), `p12_feature`, `p13_event_bus`, `p14_messaging` (index jobs), `p16_cache`, `p17_scheduler` (reindex), `p27_ai` (optional vectors)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`SEARCH_SCHEMA.md`](SEARCH_SCHEMA.md) · [`SEARCH_API.md`](SEARCH_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise search plane: indexes/mappings, pipelines, ACL trimming, facets, synonyms/analyzers, ranking, suggest, reindex/drift, query DSL safety, saved searches, hybrid/vector hooks. |
| 1.1 | 2026-09-12 | TASK-SOR-007: OpenSearch live path + `PROVIDER_PENDING` without a cluster; MEMORY test double; empty catalog `[]`; RLS on catalog/query HTTP. |

---

## 1. Purpose (enterprise)

`p18_search` is JeslotERP’s **enterprise search & indexing control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Enterprise Search / CDS search** — business-object search with authorization  
- **Microsoft Dynamics 365 / Dataverse relevance search** — multi-table search, filters, facets  
- **Salesforce SOSL / Search indexes** — cross-object query, highlighting, synonyms  
- **Elastic/OpenSearch class systems** — mappings, analyzers, aggregations, pipelines  

It is **not** `SELECT … ILIKE '%q%'` across production tables. It is the system that makes ERP findability correct for:

1. **Cross-entity search** — bilty, BP, documents, invoices, vehicles, …  
2. **Derived indexes** — source of truth remains domain DBs  
3. **Near-real-time ingestion** from p13 events + p14 chunk jobs  
4. **Security trimming** — users only see authorized hits  
5. **Facets / filters / sort** for power-user ERP grids  
6. **Locale-aware analyzers** (en/hi) via p06 hints  
7. **Synonyms, boosts, ranking profiles**  
8. **Autocomplete / typeahead**  
9. **Document & media text** (OCR/text extract refs from p08/p09)  
10. **Reindex, drift detection, query analytics**  

### Owns

| Domain | Examples |
|---|---|
| Indexes / collections | `bp.partner`, `sales.order` |
| Mappings | fields, types, facets |
| Pipelines | extract → transform → index |
| Ingestion cursors | event/job progress |
| Query profiles | DSL allow-lists, ranking |
| Synonyms / analyzers | language packs |
| Suggest indexes | autocomplete |
| Saved searches | user/tenant |
| Reindex & drift | jobs, reports |
| ACL projections | principal digests on docs |
| Governance | packs, schema activate |

### Does **not** own

| Concern | Owner |
|---|---|
| Entity/field dictionary | `p05_metadata` |
| File bytes / OCR workers | `p08_file_media` (+ jobs) |
| DMS versions | `p09_document` |
| Business writes | Domain modules |
| Vector model hosting | `p27_ai` (optional embedding provider) |
| Primary SQL reporting | `p24_reporting` |

### Critical split: Search vs Metadata vs Domain

| | **Search (p18)** | **Metadata (p05)** | **Domain** |
|---|---|---|---|
| Stores | Search documents (derived) | Shape of entities | System of record |
| Query | Relevance + facets | Not for end-user find | Transactional CRUD |
| Loss | Rebuild from source | Schema loss | Business loss |

**Rule:** Indexes are disposable; rebuild must always be possible from domain + documents.

---

## 2. Architectural position

```text
Domain TX → outbox → p13 events
                      │
                      ▼
              search pipeline (p14 jobs)
                      │
         extract via gateways (BP/order/doc/media text)
                      │
                      ▼
              index engine (OpenSearch/ES/Meili adapter)
                      │
                      ▼
              query API (ACL filter injected)
```

**Hard rules**

1. **Never** query search engine without tenant + ACL filters for user calls.  
2. No cross-schema FKs — indexed `entity_type` + `entity_id` only.  
3. Mappings versioned; breaking changes require reindex.  
4. PII fields marked; highlighting/redaction policies apply.  
5. Write path is async; read-after-write may lag (show indexing state).  
6. Query DSL is allow-listed — no scripts / raw engine queries from clients.

---

## 3. Advanced design principles

1. **Index-per-entity-type** (default) + optional alias for global search.  
2. **Mapping from metadata** — field boosts/searchable flags from p05.  
3. **Pipeline stages** — extract, normalize, enrich, acl_project, embed?, index.  
4. **Idempotent upsert** by `entity_type:entity_id:version`.  
5. **Soft-delete in index** — tombstone or remove on domain delete.  
6. **Routing key** — tenant_id for shard affinity.  
7. **Ranking profiles** — BM25 + business boosts (status, recency).  
8. **Facet dictionary** — only declared facet fields.  
9. **Synonym graphs** per locale/namespace.  
10. **Suggest** separate completion index.  
11. **Highlight** with fragment limits.  
12. **Cursor pagination** for deep pages (avoid huge `from`).  
13. **Saved searches** with sharing.  
14. **Zero-result logging** for relevance tuning.  
15. **Drift scans** — sample source vs index.  
16. **Reindex blue/green** — new index → alias swap.  
17. **Hybrid search hooks** — keyword + vector (optional).  
18. **Packs** — seed sales-order/BP/document indexes.

---

## 4. Core concepts

### 4.1 Search document

```text
index_key, entity_type, entity_id, tenant_id, company_id?,
title, subtitle, body, fields{},
acl_principals[], tags[],
locale?, updated_at, source_version, media_text_ref?
```

### 4.2 Query

```text
q, indexes[] | global,
filters{}, facets[], sort[],
locale, profile,
from/size | cursor,
highlight?, explain?
```

### 4.3 Security trimming

At index time: store `acl_principals` (user/role/group ids) and/or policy keys.  
At query time: inject filter `acl_principals IN caller_principals OR public`.  
Denial by omission — never return then redact if avoidable.

### 4.4 Ingestion modes

| Mode | Use |
|---|---|
| Event-driven | Near-real-time upserts |
| Chunk reindex | Backfill / repair |
| Full reindex | Mapping change blue/green |
| On-read repair | Rare; admin only |

### 4.5 Global search

Alias `erp.global` searches across allowed indexes with type facets (`entity_type`).

---

## 5. Analyzers & locales

- Standard analyzers per locale (`en`, `hi`)  
- Edge-ngram for suggest  
- Synonyms: `GSTIN, GST No, GST Number`  
- Stopwords optional per domain  
- ICU folding where supported  

---

## 6. Integration patterns

| Source event | Action |
|---|---|
| `bp.partner.updated` | Upsert partner doc |
| `document.released` | Index doc metadata + text extract |
| `sales.order.posted` | Upsert sales order |
| `media.ocr.ready` | Enrich attached entity body |

Indexing handlers are p14 jobs (`search.index_entity`, `search.reindex_chunk`).

---

## 7. Security

### Permissions

| Code | Use |
|---|---|
| `search.query` | User search |
| `search.suggest` | Autocomplete |
| `search.saved.manage` | Saved searches |
| `search.catalog.read` | Indexes/mappings |
| `search.catalog.manage` | Manage mappings |
| `search.reindex` | Reindex jobs |
| `search.admin` | Aliases, analyzers, purge |
| `search.analytics.read` | Query logs aggregates |
| `search.pack.install` | Packs |
| `search.*` | Wildcard |

### RLS

FORCE RLS on tenant saved searches, ingestion cursors, analytics samples.  
Index engine must still filter by tenant_id on every query.

---

## 8. Module layout

```text
platforms/p18_search/
  application/
    services/
      mapping_compiler.py
      pipeline.py
      indexer.py
      query_builder.py
      acl_filter.py
      facet_service.py
      suggest_service.py
      reindex.py
      drift.py
      ranking.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/…
    engines/ opensearch.py memory.py
    gateways/ metadata.py media.py document.py domain_*.py
  tests/unit/query/ acl/ mapping/ pipeline/
```

---

## 9. Domain events

| Event | When |
|---|---|
| `search.document.indexed` / `deleted` | Ingestion |
| `search.reindex.started` / `completed` / `failed` | Reindex |
| `search.mapping.activated` | Catalog |
| `search.drift.detected` | Ops |
| `search.alias.swapped` | Blue/green |

---

## 10. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, index catalog, permissions |
| P2 | Mapping + upsert/delete + basic query |
| P3 | ACL trimming + tenant routing |
| P4 | Facets/sort/highlight |
| P5 | Event pipelines + chunk reindex |
| P6 | Synonyms/analyzers/suggest |
| P7 | Ranking profiles + saved searches |
| P8 | Drift + blue/green + analytics |
| P9 | Optional hybrid/vector |
| P10 | Registry → **Live** |

---

## 11. Definition of Done (enterprise)

- [x] User cannot see other-tenant hits  
- [x] ACL filter denies unauthorized entity hits  
- [x] Idempotent upsert by source_version  
- [x] Reindex alias swap without downtime  
- [x] Query rejects unsafe DSL nodes  
- [x] Facet only on declared fields  
- [x] Hindi/English analyzer smoke tests  
- [x] Drift scan finds missing docs in fixture  
- [x] OpenSearch attach only after ping; pytest stays MEMORY (`PROVIDER_PENDING` if no cluster)  
- [x] Tenant RLS GUCs on catalog/query HTTP (`require_search_access`)  
- [x] Index catalog persist on `AsyncSession` (empty list is `[]`)  
- [x] No cross-schema FKs  

---

## 12. Anti-patterns

| Don’t | Do |
|---|---|
| ILIKE across live OLTP for global search | Index + query API |
| Trust client-provided filters alone | Server injects tenant+ACL |
| Index secrets / raw account numbers | Metadata sensitivity flags |
| Huge `from=100000` pagination | Cursor search_after |
| Sync index in request TX | Async pipeline |
| One giant untyped index blob | Mapped fields + entity indexes |

---

## 13. Related documents

- Schema: [`SEARCH_SCHEMA.md`](SEARCH_SCHEMA.md)  
- API: [`SEARCH_API.md`](SEARCH_API.md)  
- Metadata: [`../05_metadata/METADATA_GUIDE.md`](../05_metadata/METADATA_GUIDE.md)  
- Document: [`../09_document/DOCUMENT_GUIDE.md`](../09_document/DOCUMENT_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
