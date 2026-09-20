# JeslotERP Search Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — catalog/query HTTP + RLS. OpenSearch `POST /backends/{key}/test` is `PROVIDER_PENDING` until a cluster is attached. Not Production.  
**Package:** `platforms.p18_search`  
**PostgreSQL schema:** `search`  
**Public base:** `/api/v1/search`  
**Internal base:** `/internal/v1/search`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`SEARCH_GUIDE.md`](SEARCH_GUIDE.md) · [`SEARCH_SCHEMA.md`](SEARCH_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Query/suggest/facets, saved searches, index catalog, ingest/reindex, drift, analytics, internal index APIs, packs. |
| 1.1 | 2026-09-12 | TASK-SOR-007: backends/test + health report `PROVIDER_PENDING` without OpenSearch; catalog empty list is `[]`. |
| 1.2 | 2026-09-12 | HYG-014: `GET /changesets` + `POST /changesets/{id}/approvals` match this inventory. |

---

## 1. Design principles (advanced)

1. **Query-first for users** — `/query`, `/suggest`, `/global`.  
2. **Server injects tenant + ACL** — client filters cannot widen scope.  
3. **Allow-listed query DSL** — no scripts/raw engine queries.  
4. **Cursor pagination** for deep result sets.  
5. **Async indexing** — API returns accepted; consistency eventual.  
6. **Blue/green reindex** via aliases.  
7. **Explain optional** and permissioned.  
8. **PII-aware highlight**.  
9. **Idempotent reindex/ingest**.  
10. **Internal extract/index** for workers only.  
11. **Global search** respects per-index ACL.  
12. **Analytics sampled** — not full verbatim forever.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Company-Id: <uuid>
X-Internal-Token: <token>
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {
    "request_id": "…",
    "took_ms": 42,
    "total_hits": 128,
    "cursor": "…"
  }
}
```

---

## 4. Errors

```text
SRCH_INDEX_NOT_FOUND / MAPPING_NOT_FOUND / PROFILE_NOT_FOUND
SRCH_QUERY_INVALID / DSL_NODE_DENIED / SIZE_LIMIT
SRCH_FILTER_DENIED / ACL_DENIED
SRCH_FACET_NOT_ALLOWED / SORT_NOT_ALLOWED
SRCH_SUGGEST_UNAVAILABLE
SRCH_REINDEX_RUNNING / REINDEX_FAILED / SWAP_CONFLICT
SRCH_INGEST_FAILED / EXTRACTOR_UNKNOWN
SRCH_DRIFT_SCAN_RUNNING
SRCH_SAVED_NOT_FOUND
SRCH_BACKEND_UNHEALTHY
SRCH_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
SRCH_PACKAGE_CHECKSUM_MISMATCH
SRCH_EXPLAIN_DENIED
SRCH_HYBRID_UNAVAILABLE
```

HTTP: `404` · `409` · `422` · `403` · `429` · `503`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `search.query` | Query |
| `search.suggest` | Suggest |
| `search.saved.manage` | Saved searches |
| `search.catalog.read` / `manage` | Catalog |
| `search.reindex` | Reindex |
| `search.admin` | Aliases/analyzers |
| `search.analytics.read` | Analytics |
| `search.pack.install` | Packs |
| `search.*` | All |

---

## 6. Query APIs (primary)

### 6.1 Search

```http
POST /api/v1/search/query
```

```json
{
  "q": "MH12AB1234 POD delay",
  "indexes": ["sales.order", "document.document"],
  "filters": {
    "company_id": "…",
    "status": ["POSTED", "IN_TRANSIT"],
    "updated_at": { "gte": "2026-09-01T00:00:00Z" }
  },
  "facets": ["status", "branch_id", "entity_type"],
  "sort": [{ "field": "updated_at", "order": "desc" }],
  "locale": "en",
  "profile": "erp_default",
  "size": 25,
  "cursor": null,
  "highlight": true,
  "explain": false
}
```

**Response hits:**

```json
{
  "hits": [
    {
      "index_key": "sales.order",
      "entity_type": "sales.order",
      "entity_id": "…",
      "score": 12.4,
      "title": "BL/MH01/2526/NDL/000148",
      "subtitle": "Acme → Beta · IN_TRANSIT",
      "fields": { "vehicle_no": "MH12AB1234", "status": "IN_TRANSIT" },
      "highlight": { "body": ["… <em>MH12AB1234</em> …"] }
    }
  ],
  "facets": {
    "status": [{ "value": "IN_TRANSIT", "count": 40 }]
  },
  "cursor": "eyJ…"
}
```

Tenant + ACL filters always applied server-side.

### 6.2 Global search

```http
POST /api/v1/search/global
```

Same body without indexes (uses `erp.global` alias / allowed indexes). Facet `entity_type` recommended.

### 6.3 Suggest / autocomplete

```http
GET /api/v1/search/suggest?q=Acm&index=bp.partner&limit=10
POST /api/v1/search/suggest
```

```json
{
  "q": "Acm",
  "indexes": ["bp.partner"],
  "limit": 10
}
```

Returns lightweight suggestions `{ text, entity_id, entity_type }`.

### 6.4 Multi-get by entity

```http
POST /api/v1/search/mget
```

```json
{
  "docs": [
    { "index_key": "sales.order", "entity_id": "…" }
  ]
}
```

Still ACL-checked.

### 6.5 Explain (admin/debug)

```http
POST /api/v1/search/explain
```

Requires profile `allow_explain` + permission; returns ranking breakdown.

---

## 7. Saved searches

```http
GET    /api/v1/search/saved
POST   /api/v1/search/saved
GET    /api/v1/search/saved/{id}
PATCH  /api/v1/search/saved/{id}
DELETE /api/v1/search/saved/{id}
POST   /api/v1/search/saved/{id}/execute
POST   /api/v1/search/saved/{id}/shares
```

---

## 8. Catalog — indexes & mappings

```http
GET  /api/v1/search/indexes
POST /api/v1/search/indexes
GET  /api/v1/search/indexes/{index_key}
GET  /api/v1/search/indexes/{index_key}/mappings
POST /api/v1/search/indexes/{index_key}/mappings
POST /api/v1/search/mappings/{mapping_id}/activate
GET  /api/v1/search/aliases
POST /api/v1/search/aliases/{alias_key}/points-to
```

Activating a breaking mapping usually requires reindex job (policy).

---

## 9. Analyzers, synonyms, ranking

```http
GET  /api/v1/search/analyzers
PUT  /api/v1/search/synonyms/{set_key}
GET  /api/v1/search/rank-profiles
PUT  /api/v1/search/rank-profiles/{profile_key}
PUT  /api/v1/search/query-profiles/{profile_key}
```

**Synonyms put:**

```json
{
  "locale": "en",
  "entries": [
    { "from": ["GSTIN", "GST No"], "to": "GSTIN" }
  ]
}
```

---

## 10. Ingestion & reindex

### 10.1 Enqueue index entity (internal / trusted)

```http
POST /internal/v1/search/index
POST /internal/v1/search/delete
```

```json
{
  "index_key": "sales.order",
  "entity_id": "…",
  "source_version": "2026-09-09T04:00:00Z",
  "payload": { "optional": "pre-extracted doc" }
}
```

If payload omitted, pipeline EXTRACT runs via gateway.

### 10.2 Reindex job

```http
POST /api/v1/search/reindex
GET  /api/v1/search/reindex/{job_id}
POST /api/v1/search/reindex/{job_id}/cancel
```

```json
{
  "index_key": "sales.order",
  "mapping_id": "…",
  "swap_alias": true,
  "chunk_size": 500
}
```

Creates BUILDING physical index → chunk jobs → alias swap → retire old.

### 10.3 ACL rebuild

```http
POST /api/v1/search/indexes/{index_key}/acl-rebuild
```

### 10.4 Pipeline / bindings

```http
GET /api/v1/search/pipelines
PUT /api/v1/search/pipelines/{pipeline_key}/stages
PUT /api/v1/search/source-bindings
```

---

## 11. Drift

```http
POST /api/v1/search/drift-scans
GET  /api/v1/search/drift-scans/{id}
GET  /api/v1/search/drift-scans/{id}/findings
POST /api/v1/search/drift-scans/{id}/repair
```

Repair enqueues index/delete for findings.

---

## 12. Hybrid (optional)

```http
POST /api/v1/search/hybrid
```

```json
{
  "q": "damaged goods claim",
  "indexes": ["document.document"],
  "hybrid_profile": "docs_default",
  "size": 20
}
```

Fails with `SRCH_HYBRID_UNAVAILABLE` if vectors not configured.

---

## 13. Analytics

```http
GET /api/v1/search/analytics/top-queries?from=…&to=…
GET /api/v1/search/analytics/zero-results
GET /api/v1/search/analytics/latency
```

Requires `search.analytics.read`.

---

## 14. Packages & governance

```http
GET  /api/v1/search/packages
POST /api/v1/search/packages/{package_key}/install
GET  /api/v1/search/changesets
POST /api/v1/search/changesets/{id}/approvals
```

---

## 15. Engine admin

```http
GET  /api/v1/search/backends
POST /api/v1/search/backends/{backend_key}/test
GET  /api/v1/search/health
```

`opensearch_primary` test/health stay `PROVIDER_PENDING` (no invented hits) until `try_attach_opensearch()` pings a reachable URL. Pytest never attaches. `MEMORY` remains the TestClient double.

---

## 16. Worker ticks (internal)

```http
POST /internal/v1/search/ingest/tick
POST /internal/v1/search/reindex/chunks/{chunk_id}/run
POST /internal/v1/search/pipelines/execute
```

Invoked by p14 handlers / event bridges.

---

## 17. Caching & concurrency

| Resource | Strategy |
|---|---|
| Query profiles / mappings | Cached; invalidate on activate |
| Suggest | Short TTL cache optional |
| Reindex | Single active job per index |
| Upsert | Idempotent on source_version |

---

## 18. Example flows

### 18.1 User finds a sales order by vehicle

1. `POST /query` indexes `sales.order` q=`MH12…`  
2. ACL + tenant filters applied  
3. Open hit → domain get by `entity_id`  

### 18.2 Document released

1. p13 `document.released.v1` → bridge → `search.index_entity` job  
2. Extract metadata + text via p09/p08  
3. Upsert into `document.document`  

### 18.3 Mapping change

1. New mapping DRAFT → approve  
2. `POST /reindex` swap_alias=true  
3. Alias cutover; old index retired  

### 18.4 Drift repair

1. Nightly drift scan  
2. MISSING_IN_INDEX findings → repair enqueue  

---

## 19. Event hooks

| Event | Consumer |
|---|---|
| `search.reindex.completed` | Notify admin |
| `search.drift.detected` | Ops |
| `search.mapping.activated` | Cache bust query profiles |
| `search.alias.swapped` | Monitoring |

---

## 20. Compatibility notes

- Public prefix `/api/v1/search`; schema `search`.  
- Clients must not assume immediate read-your-writes; optional `wait_for` internal only.  
- Reporting megascan remains p24 — search is findability, not warehouse.  
- Vector features optional and off by default.

---

## 21. Related documents

- Guide: [`SEARCH_GUIDE.md`](SEARCH_GUIDE.md)  
- Schema: [`SEARCH_SCHEMA.md`](SEARCH_SCHEMA.md)  
- Metadata: [`../05_metadata/METADATA_API.md`](../05_metadata/METADATA_API.md)  
- Document: [`../09_document/DOCUMENT_API.md`](../09_document/DOCUMENT_API.md)  
- Event bus: [`../13_event_bus/EVENT_BUS_API.md`](../13_event_bus/EVENT_BUS_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
