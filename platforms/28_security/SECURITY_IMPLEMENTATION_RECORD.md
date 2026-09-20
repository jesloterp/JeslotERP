# JeslotERP Security Platform (p28) — Implementation Record

**Date:** 2026-09-12  
**Package:** `platforms.p28_security`  
**PostgreSQL schema:** `security`  
**Source of truth reviewed:** `SECURITY_GUIDE.md`, `SECURITY_SCHEMA.md`, `SECURITY_API.md`, `docs/sample.md`

---

## 1. Overview & Objective

Deliver the security control plane: KMS key *metadata*, rotation, policy/posture/WAF contract, and control events. Do not own login (p01), secret *values* (p03), or SIEM (p19). Lean 8+2 tables. Status **SoR-Live**, not Production.

## 2. All 3 Source Documents Reviewed

| Document | Version | Review outcome |
| --- | --- | --- |
| `SECURITY_GUIDE.md` | 1.0 | Ownership split + DoD mapped |
| `SECURITY_SCHEMA.md` | 1.0 | 8+2 tables; Alembic `f28a` / `f28b` |
| `SECURITY_API.md` | 1.0 | Keys + errors + `security.*` |

## 3. Existing Backend Architecture Reviewed

Mirrored p04 access deps + p12 ModulePlugin/Alembic pair + p15 named facade. HTTP does not call a production CatalogStore.

## 4. Requirements Identified

Full RTM in `SECURITY_RTM.md`.

## 5. Requirement-by-Requirement Implementation

| Area | Implementation | Verification |
| --- | --- | --- |
| Key lifecycle | `SecurityService` + `LocalDevKmsAdapter` | no-secret GET |
| Pending KMS | `PendingKmsAdapter` | `test_kms_port` |
| HTTP RLS | `require_sec_access` | auth matrix |
| Persist | `persist_row` if `AsyncSession` | durable SoR |
| DDL + RLS | `f28a0b1c2d3e` / `f28b1c2d3e4f` | module + RLS case |

## 6. Files/Modules/Services Created or Modified

**Platform:** `platforms/p28_security/**` — domain, `SecurityService`, KMS ports, ORM, public/internal routers, `require_sec_access`.  
**Wiring:** `apps/api/main.py`, `alembic/env.py`, `tests/integration/rls/test_platform_rls_isolation.py`.  
**Docs:** GUIDE/SCHEMA/API + this record + `SECURITY_RTM.md`.

## 7. Database Changes & Migrations

| Revision | Purpose |
| --- | --- |
| `f28a0b1c2d3e` | CREATE SCHEMA `security`; 10 tables |
| `f28b1c2d3e4f` | ENABLE + FORCE RLS |

Chain: `f27b1c2d3e4f` → `f28a` → `f28b`.

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/security`: providers, keys CRUD/rotate/retire, policy, posture, WAF, events. Internal `/internal/v1/security`.

## 9. Business Rules & Workflows Implemented

- GET strips `wrapped_ref` / `_secret_plain`  
- Unknown provider → `422 SEC_PROVIDER_UNKNOWN`  
- Rotate in-flight → `409 SEC_ROTATION_IN_FLIGHT`  
- LOCAL_DEV ACTIVE; AWS_KMS / AZURE_KV / HSM `PROVIDER_PENDING`

## 10. Validation, Permissions & Error Handling

`SecurityError` codes per API §4. Permissions `security.*`. `require_sec_access` applies RLS GUC then permission.

## 11. Integrations Implemented

Depends on p01 (authz). Secret values stay p03. Events intended for p19 via outbox `jesloterp:security:outbox`.

## 12. Test Cases Created for Each Functionality

| File | Coverage |
| --- | --- |
| `test_security_api_contracts.py` | no-secret, unknown provider, rotate, auth |
| `test_kms_port.py` | LOCAL_DEV wrap + pending honesty |
| `test_permissions_gate.py` | `security.*` |
| `test_module_tables.py` | 10 tables + load order |
| `test_durable_sor.py` | persist skip-mock / FakeAsyncSession |

## 13. Test Execution Results

See continue-pass pytest in `docs/task/PLATFORM-P28-P33-FINAL-REPORT.md`.

## 14. Requirements Traceability Matrix (RTM)

See `SECURITY_RTM.md`. Cross-platform: `docs/task/PLATFORM-P28-P33-RTM.md`.

## 15. Issues Found & How They Were Resolved

| Issue | Resolution |
| --- | --- |
| Mixin `version` vs business version (other platforms) | p28 keys use `current_version` |
| CatalogStore temptation | Named `SecurityService` |

## 16. Regression/Existing Functionality Verification

p05–p27 uncommitted SoR tree left untouched. Module load asserts p01 appears before p28 (topo), not “after p27”.

## 17. Final Coverage & Completion Status

| Deliverable | Status |
| --- | --- |
| Package + HTTP + ports | Complete |
| 10 ORM tables | Complete |
| Alembic + RLS cases | Complete |
| Per-platform RTM + record | Complete |
| Production label | **Withheld** |

## 18. Remaining Issues or Limitations

1. P28-LIVE-001: named AWS/Azure/HSM adapters exist. Without credentials / under pytest they are `PROVIDER_PENDING` and never invent wrap handles. Live encrypt is not claimed.  
2. `list_keys` / `get_key` are DB-first. Policies/posture/WAF/events still use the test-double ledger.  
3. RLS HTTP suite skips without Postgres.  
4. AUD-008: p03 secret rows are ciphertext (P28-LIVE-002). Live KMS wrap of those payloads is still a port. p23 webhook `_secret_plain` is out of this slice.
