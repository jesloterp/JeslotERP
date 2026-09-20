# JeslotERP Extensibility Platform (p29) — Implementation Record

**Date:** 2026-09-12  
**Package:** `platforms.p29_extensibility`  
**PostgreSQL schema:** `extensibility`  
**Source of truth reviewed:** `EXTENSIBILITY_GUIDE.md`, `EXTENSIBILITY_SCHEMA.md`, `EXTENSIBILITY_API.md`, `docs/sample.md`

---

## 1. Overview & Objective

Allow-listed hook pipeline (bind / activate / execute). No `eval`/`exec`. Generic `resource_type` only. Status **SoR-Live**.

## 2. All 3 Source Documents Reviewed

| Document | Version | Review outcome |
| --- | --- | --- |
| `EXTENSIBILITY_GUIDE.md` | 1.0 | Pipeline + fail-closed + no eval |
| `EXTENSIBILITY_SCHEMA.md` | 1.0 | 6+2 tables; `f29a` / `f29b` |
| `EXTENSIBILITY_API.md` | 1.0 | Bind/execute + `extensibility.*` |

## 3. Existing Backend Architecture Reviewed

p12 ModulePlugin pair; handler port like p16; HTTP via named `PipelineService`.

## 4. Requirements Identified

Full RTM in `EXTENSIBILITY_RTM.md`.

## 5. Requirement-by-Requirement Implementation

| Area | Implementation | Verification |
| --- | --- | --- |
| Allow-list | `PipelineService.allow` | unknown / denied |
| Handlers | `NoopHandler` / `FailHandler` | execute + fail-closed |
| Priority | sort before run | priority order test |
| No eval | builtin registry only | `test_no_eval` |
| HTTP RLS | `require_ext_access` | auth matrix |

## 6. Files/Modules/Services Created or Modified

`platforms/p29_extensibility/**`, Alembic `f29*`, `apps/api/main.py`, `alembic/env.py`, RLS cases, this record + RTM.

## 7. Database Changes & Migrations

| Revision | Purpose |
| --- | --- |
| `f29a0b1c2d3e` | SCHEMA `extensibility`; 8 tables |
| `f29b1c2d3e4f` | FORCE RLS |

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/extensibility`: points, handlers, bindings, activate/deactivate, execute, executions. Internal `/internal/v1/extensibility`.

## 9. Business Rules & Workflows Implemented

- Unknown handler → `422 EXT_HANDLER_UNKNOWN`  
- Not allow-listed → `403 EXT_HANDLER_DENIED`  
- Inactive skipped  
- FAIL_CLOSED → `409 EXT_FAIL_CLOSED` (later handlers do not succeed)  
- `ext.sample.fail` registered, not default-allow-listed

## 10. Validation, Permissions & Error Handling

`ExtensibilityError` + `require_ext_access`. Permissions `extensibility.*`.

## 11. Integrations Implemented

Not yet invoked from p04/p05/p10. Outbox `jesloterp:extensibility:outbox`.

## 12. Test Cases Created for Each Functionality

| File | Coverage |
| --- | --- |
| `test_extensibility_api_contracts.py` | noop, unknown, inactive, fail-closed, priority, auth |
| `test_no_eval.py` | bans eval/exec/compile outside tests |
| `test_permissions_gate.py` / `test_module_tables.py` / `test_durable_sor.py` | gates, 8 tables, persist |

## 13. Test Execution Results

See `docs/task/PLATFORM-P28-P33-FINAL-REPORT.md`.

## 14. Requirements Traceability Matrix (RTM)

See `EXTENSIBILITY_RTM.md`.

## 15. Issues Found & How They Were Resolved

| Issue | Resolution |
| --- | --- |
| Fail handler default-open risk | Registered, not allow-listed |
| Timeout DoD | `EXT_TIMEOUT` via isolated child process; overtime `terminate` then `kill` |

## 16. Regression/Existing Functionality Verification

p04 create/update/delete, p05 publish, and p10 start/cancel call `invoke_hooks` (no-op unless a binding is ACTIVE).

## 17. Final Coverage & Completion Status

Documented v1 surface SoR-Live. Timeout + consumer hooks added 2026-09-12.

## 18. Remaining Issues or Limitations

1. Isolation is a child process (`spawn`), not a container/seccomp sandbox.  
2. Windows spawn is slow; a tight `timeout_ms` may fire before the child starts — still `EXT_TIMEOUT` + kill.  
3. Allow-listed builtins only. No tenant-uploaded code. Not Production.  
4. Consumers wired: p04 create/update/delete, p05 publish (before+after), p10 start/cancel. Activate/blacklist/signal/complete_task/apply_changeset are not. No CRM.  
5. RLS suite skips without Postgres.  
6. K28-33-001: list bindings/executions are DB-first. Points/handlers still memory.
