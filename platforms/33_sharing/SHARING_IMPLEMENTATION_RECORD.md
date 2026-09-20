# JeslotERP Sharing Platform (p33) — Implementation Record

**Date:** 2026-09-12  
**Package:** `platforms.p33_sharing`  
**PostgreSQL schema:** `sharing`  
**Source of truth reviewed:** `SHARING_GUIDE.md`, `SHARING_SCHEMA.md`, `SHARING_API.md`, `docs/sample.md`

---

## 1. Overview & Objective

Record-level ACL evaluate (owner / manual / team / rule / hierarchy). Tenant RLS is not record share. Generic `resource_type` + UUID only. Status **SoR-Live**.

## 2. All 3 Source Documents Reviewed

| Document | Version | Review outcome |
| --- | --- | --- |
| `SHARING_GUIDE.md` | 1.0 | Evaluate chain + stranger deny |
| `SHARING_SCHEMA.md` | 1.0 | 6+2 tables |
| `SHARING_API.md` | 1.0 | Teams/grants/evaluate |

## 3. Existing Backend Architecture Reviewed

Named `SharingService`; `require_shr_access`; no CRM tables.

## 4. Requirements Identified

Full RTM in `SHARING_RTM.md`.

## 5. Requirement-by-Requirement Implementation

| Area | Implementation | Verification |
| --- | --- | --- |
| Owner / stranger | evaluate | owner true / stranger false |
| Team | membership + TEAM grant | team reason |
| Hierarchy | ancestor ROLE grant | hierarchy test |
| Idempotent grant | same grant_id | duplicate POST |

## 6. Files/Modules/Services Created or Modified

`platforms/p33_sharing/**`, Alembic `f33*`, wiring, this record + RTM.

## 7. Database Changes & Migrations

| Revision | Purpose |
| --- | --- |
| `f33a0b1c2d3e` | SCHEMA `sharing`; 8 tables |
| `f33b1c2d3e4f` | FORCE RLS |

## 8. APIs/Endpoints Implemented or Updated

Public `/api/v1/sharing`: teams/members, hierarchy, rules, grants, evaluate. Internal `/internal/v1/sharing`.

## 9. Business Rules & Workflows Implemented

Evaluate order OWNER → MANUAL → TEAM → RULE → HIERARCHY. Duplicate grant returns the same `grant_id`.

## 10. Validation, Permissions & Error Handling

`SharingError` + `sharing.*` + `require_shr_access`.

## 11. Integrations Implemented

Not yet called from domain read paths. Outbox `jesloterp:sharing:outbox`.

## 12. Test Cases Created for Each Functionality

| File | Coverage |
| --- | --- |
| `test_sharing_api_contracts.py` | owner/team/stranger, idempotent, hierarchy, auth |
| permissions / module / durable SoR | gates, 8 tables, persist |

## 13. Test Execution Results

See `docs/task/PLATFORM-P28-P33-FINAL-REPORT.md`.

## 14. Requirements Traceability Matrix (RTM)

See `SHARING_RTM.md`.

## 15. Issues Found & How They Were Resolved

| Issue | Resolution |
| --- | --- |
| Tenant-id-as-ACL temptation | Explicit evaluate API |

## 16. Regression/Existing Functionality Verification

No sales-order share tables. p05–p27 untouched.

## 17. Final Coverage & Completion Status

Evaluate v1 SoR-Live. HTTP/internal evaluate is DB-first (P33-LIVE-001). p04 public partner-scoped reads + list consume sharing (opt-in when grants exist).

## 18. Remaining Issues or Limitations

1. p04 public partner-scoped reads use `require_partner_share` (GET + validate-for-use). No grants → skip. Create writes OWNER (`commit=False`). Domain reads still `evaluate_sync` (memory).  
2. HTTP/internal `POST /evaluate` and list grants/teams are DB-first. p04 list share-filters page-local. Writes are not share-gated. Hierarchy/rules still memory.  
3. Internal `/internal/v1/bp` GET bypasses sharing (system path). Not CRM.  
4. RULE fixtures remain thin. RLS suite skips without Postgres.
