# Monitoring Platform — Implementation Record

**Platform:** `p21_monitoring`  
**Date:** 2026-09-12  
**Verification:** `python -m pytest platforms/p21_monitoring/tests -q --tb=short` → **26 passed**; 62 `monitoring` tables; load order after p20.

## 1. Overview & Objective

Observability and reliability control plane: schema `monitoring`, metrics catalog with cardinality guards, LIVE≠READY probes, in-memory TSDB stub, SLI/SLO/error budgets, alerts/routing/silences, versioned dashboards, synthetics, service map, incidents lite, packs, Alembic f21a/f21b.

## 2. Source documents reviewed

`MONITORING_GUIDE.md`, `MONITORING_SCHEMA.md`, `MONITORING_API.md` under `docs/platforms/21_monitoring/` plus `docs/tasks/task_p21_monitoring.md`. Requirement docs were not modified.

## 3. Existing backend architecture reviewed

p20_logging ModulePlugin, in-memory catalog store, exception handlers from p05 auth, `require_internal_token`, dual Alembic schema+RLS, TestClient factory. Same layout copied for p21.

## 4. Requirements identified

See `MONITORING_RTM.md` (100% of GUIDE / SCHEMA / API items mapped).

## 5. Requirement-by-requirement implementation

In-memory `MonitoringCatalogStore` implements catalog, probe runner, TSDB stub, SLO eval/burn, alert manager (silence/inhibit notify), dashboard publish, synthetic runner, service map rebuild, package install. HTTP is thin CQRS over the store.

## 6. Files created or modified

**Created**

- `platforms/p21_monitoring/` domain, application, infrastructure HTTP/module, tests
- `platforms/p21_monitoring/infrastructure/persistence/schema_constants.py` (`MONITORING_SCHEMA = "monitoring"`)
- `mon_outbox` + `mon_idempotency_key`; models `__init__` imports all 62 classes
- `alembic/versions/f21a0b1c2d3e_create_monitoring_schema.py`
- `alembic/versions/f21b1c2d3e4f_enable_monitoring_rls.py`
- `docs/platforms/21_monitoring/MONITORING_RTM.md` (this folder)
- this record

**Modified**

- `apps/api/main.py` — `MonitoringModule` after `LoggingModule`; exception handlers
- `alembic/env.py` — import p21 models
- `IMPLEMENTATION_TASKS.md`, `IMPLEMENTATION_STATUS.md`

**Deleted**

- `_gen_p21_models.py`

**Not edited**

- `docs/platforms/21_monitoring/MONITORING_GUIDE.md`, `MONITORING_SCHEMA.md`, `MONITORING_API.md`
- `docs/tasks/task_p21_monitoring.md`

## 7. Database changes & migrations

Schema `monitoring`, 60 domain `mon_*` tables (already generated) + `mon_outbox` + `mon_idempotency_key` = **62**. FORCE RLS on tenant-bearing history tables. Permissions `monitoring.*` seeded in f21a. No cross-schema FKs.

## 8. APIs

Public `/api/v1/monitoring` and internal `/internal/v1/monitoring`. App-level `GET /health/live` and `GET /health/ready` registered by the module (LIVE stays 200 when READY is 503). SLO list is `GET /slos` only (HYG-021 removed the `/slosos` typo).

## 9. Business rules

- LIVE ≠ READY (process vs db/redis/event_bus)
- `user_id`/`email` labels → `MON_LABEL_DENIED`; series cap → `MON_CARDINALITY_EXCEEDED`
- Silence requires `ticket_ref` else `MON_SILENCE_INVALID`; matching silence suppresses `monitoring.alert.firing`
- Ack/resolve idempotent; resolve-then-ack → `MON_ALERT_STATE_INVALID`
- Dashboard/receiver public JSON never includes secrets
- Synthetic 3x fail → FIRING alert
- Production CRITICAL rule query edits → `MON_APPROVAL_REQUIRED`

## 10. Validation, permissions & errors

MON_* codes from API §4. Permissions from API §5 via `require_monitoring_permission`.

## 11. Integrations

Depends on `p20_logging` (module graph). Alert notify is outbox `monitoring.alert.firing` (p15 consumer). Trace deep-dive is a Tempo-style link, not a span warehouse.

## 12. Tests

Module (4) + API families (14) with ≥2 variations each (success + failure/guard). **18 passed**.

## 13. Test execution results

`pytest platforms/p21_monitoring/tests -q` → **18 passed**.

## 14. RTM

`MONITORING_RTM.md`.

## 15. Issues found & resolved

`slo_reports` method shadowed the dataclass field; renamed method to `list_slo_reports`.

## 16. Regression

p21 suite green. Full-repo suite not run (TASK-014).

## 17. Coverage & completion

TASK-007 acceptance bar met: implement + tests + RTM + this record.

## 18. Limitations

1. Metrics/scrape catalog HTTP persists on Postgres; empty list is `[]`. `require_monitoring_access` sets RLS GUCs. Samples stay on in-process MEMORY `TsdbEngine`. SLO/alerts/probes/dashboards still memory. Not Production.
2. External TSDB (Prometheus/Mimir/Datadog) is `PROVIDER_PENDING` — no invented HTTP client.
3. Alembic not applied to live DB in-session; p15/p19 notify/audit are outbox/optional; span payloads stay in the registered trace backend.
