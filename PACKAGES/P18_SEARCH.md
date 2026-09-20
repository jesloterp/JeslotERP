# Search (`p18_search`)

**Package:** `p18_search`  
**Schema:** `search`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Search and index control plane: mappings, ACL-trimmed query, suggest, reindex / drift, and saved searches.

## 2. Responsibilities

- Own the `search` persistence schema and the `p18_search` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Index / mapping / alias / analyzer / pipeline catalog
- Query, suggest, saved, and hybrid APIs
- Reindex, drift, analytics, backends
- Internal index / delete / ingest tick
- In-memory engine for isolated tests
- External search-engine adapter attaches when reachable

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Index**
- **Mapping version**
- **Alias**
- **Analyzer**
- **Rank / query profile**
- **Saved search**
- **ACL principal**
- **Reindex job**
- **Drift scan**

## 5. Major Capabilities

- Index / mapping / alias / analyzer / pipeline catalog
- Query, suggest, saved, and hybrid APIs
- Reindex, drift, analytics, backends
- Internal index / delete / ingest tick
- In-memory engine for isolated tests
- External search-engine adapter attaches when reachable

## 6. Public Architecture

```text
HTTP / internal API
        ↓
Application services (commands, queries, ports)
        ↓
Domain concepts and policies
        ↓
Adapters (persistence, optional providers, outbox)
```

Conceptual tables (purpose only):

- Indexes / mappings
- ACL / query profiles
- Pipelines / analysis
- Saved searches / jobs
- Governance

Public API resource groups:

- Catalog
- Query / suggest / saved
- Ops
- Internal ingest

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p05_metadata`
- `p08_file_media`
- `p09_document`

### Additional verified runtime coupling

- Identity permission seed; auth contract shared with the metadata plane

### Recommended future dependency

- p27_ai vector / hybrid
- p14_messaging for reindex jobs
- p01_identity ACL principals

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Reporting
- AI corpora / chunk index
- future global find

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `search.document.indexed`
- `search.mapping.activated`
- `search.reindex.completed`
- `search.drift.detected`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Engine attach is optional at startup.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- search permissions
- Unsafe query-node denial
- Facet ACL
- RLS

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Search-engine port
- Internal indexing API

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `search` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Production cluster operations are not claimed
- Default isolated tests use the memory engine

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Operators must find partners, documents, and postings with the same ACL they have on the record.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
