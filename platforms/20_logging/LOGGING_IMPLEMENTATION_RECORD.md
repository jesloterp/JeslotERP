# Logging Platform — Implementation Record

**Platform:** `p20_logging`  
**Date:** 2026-09-12  
**Verification:** `python -m pytest platforms/p20_logging/tests -q --tb=short` → **21 passed**; 60 `logging` tables.

## Overview

Structured operational logging control plane: schema `logging`, ingest/OTLP, PII scrub, hot query/tail, time-boxed DEBUG overrides, fingerprints, pipelines/sinks, shippers, signals, Alembic f20a/f20b. TASK-SOR-018: hot path + catalog HTTP on Postgres; honest sink port.

## Sources

`LOGGING_GUIDE.md`, `LOGGING_SCHEMA.md`, `LOGGING_API.md`.

## Architecture

HTTP → persist/fetch only when `session` is `AsyncSession`. TestClient/`AsyncMock` stays the memory double. Empty catalog/query is `[]` / `{items:[],total:0}`. `require_logging_access` sets RLS GUCs on durable routes.

## Implementation

- 60 ORM tables (`log_*` + outbox + idempotency)
- `LoggingCatalogStore` — schema validation, scrub, rate-drop INFO first, ERROR fingerprints, 24h query guard
- HTTP `/api/v1/logging` + `/internal/v1/logging`
- `LoggingModule` depends on `p01_identity`; wired in `apps/api/main.py` and `alembic/env.py`
- `SinkPort`: STDOUT in-process `ACTIVE`; OTLP/ELASTIC/LOKI/CLOUDWATCH/FILE → `PROVIDER_PENDING`

## Limitations

1. Hot ingest/query/catalog HTTP persists on Postgres; empty list is `[]`. Tail, fingerprints, shippers, signals, stats, packages still memory. Not Production.
2. External sinks are ports only — no invented Splunk/ELK/CloudWatch clients.
3. Internal OTLP ingest stays memory.

## Completion

TASK-SOR-018 verified. Registry **SoR-Live**.
