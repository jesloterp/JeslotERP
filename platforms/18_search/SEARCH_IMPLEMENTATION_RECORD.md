# Search Platform — Implementation Record

**Platform:** `p18_search`  
**Date:** 2026-09-12  
**Verification:** `pytest platforms/p18_search/tests -q` → **29 passed**; 62 `search` tables; load order after p17.

## Overview

Enterprise search control plane: schema `search`, MEMORY engine adapter, ACL/tenant trim, query/suggest/global, saved searches, mapping activate, blue/green reindex, drift repair, packs, permissions, Alembic f18a/f18b.

## Sources

GUIDE / SCHEMA / API under `docs/platforms/18_search/` plus `docs/tasks/task_p18_search.md`. Requirement docs not modified.

## Architecture reviewed

p17/p16 ModulePlugin, in-memory store, exception handlers, RLS pair, no cross-schema FKs.

## Implementation

- Domain enums/errors; 62 ORM models (`catalog/analysis/pipeline/acl/query/analytics/jobs/governance` + outbox/idempotency)
- `SearchCatalogStore` — query DSL allow-list, ACL, facets, cursor, upsert, reindex swap, drift
- HTTP `/api/v1/search` + `/internal/v1/search`
- `SearchModule` deps p05+p08+p09; wired in `apps/api/main.py` and `alembic/env.py`

## Limitations

OpenSearch attaches only when `OPENSEARCH_URL` (or backend endpoint) pings; otherwise `PROVIDER_PENDING`. MEMORY is the pytest/TestClient double — not a live cluster. Extract gateways stubbed; hybrid off; no OpenSearch soak. Not Production.

## Tests

Module (4), query (2), ACL/drift (2), API contracts (7 groups).
