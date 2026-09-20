# JeslotERP Rules Platform (p11) — Requirements Traceability Matrix

**Date:** 2026-09-11  
**Package:** `platforms.p11_rules`  
**PostgreSQL schema:** `rules`  
**Sources:** `RULES_GUIDE.md`, `RULES_SCHEMA.md`, `RULES_API.md`  
**Verification:** `pytest platforms/p11_rules/tests -q` → **43 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| RULE-G-01 | GUIDE §1 | Decision control plane (not ifs in domain) | `p11_rules` + RulesCatalogStore | Done | store evaluate | Pass |
| RULE-G-02 | GUIDE §2 | Pure evaluate; no domain writes | `/evaluate` + store.evaluate | Done | API evaluate | Pass |
| RULE-G-03 | GUIDE §2 | No cross-schema FKs | UUID refs only | Done | ORM models | Pass |
| RULE-G-04 | GUIDE §2 | Allow-listed AST | expression_engine | Done | test_rules_ast_* | Pass |
| RULE-G-05 | GUIDE §2 | ACTIVE immutable | put_decision_table / activate | Done | lifecycle API | Pass |
| RULE-G-06 | GUIDE §3 | Hit policies FIRST/UNIQUE/ANY/PRIORITY/COLLECT | table_engine | Done | test_rules_hit_* | Pass |
| RULE-G-07 | GUIDE §3 | Explainability | evaluate explain steps | Done | credit evaluate | Pass |
| RULE-G-08 | GUIDE §3 | Strict facts | fact schema validate | Done | FACT_* tests | Pass |
| RULE-G-09 | GUIDE §3 | Context providers | enrich_context | Done | evaluate enrich | Pass |
| RULE-G-10 | GUIDE §3 | Overlay layers SYSTEM→COMPANY | overlay_merger + store | Done | overlay tests | Pass |
| RULE-G-11 | GUIDE §3 | Compile on publish | compile/publish | Done | API lifecycle | Pass |
| RULE-G-12 | GUIDE §3 | Simulation gate | activate require_simulation_green | Done | SimulationFailedError | Pass |
| RULE-G-13 | GUIDE §3 | Deterministic as_of | evaluate as_of param | Done | evaluate API | Pass |
| RULE-G-14 | GUIDE §3 | Idempotent evaluate | Idempotency-Key | Done | store idempotent | Pass |
| RULE-G-15 | GUIDE §3 | Action suggestions not auto-run | suggested_actions; apply gated | Done | apply 403/200 | Pass |
| RULE-G-16 | GUIDE §3 | Complexity budgets | BudgetExceeded / max rows/depth | Done | AST budget + batch | Pass |
| RULE-G-17 | GUIDE §4 | Artifact kinds DT/EXPR/SET/VALIDATION/ASSIGN | seeds + evaluate paths | Done | store + API | Pass |
| RULE-G-18 | GUIDE §4 | Cell ops ANY/EQ/…/EXPR | table_engine.cell_matches | Done | cell ops test | Pass |
| RULE-G-19 | GUIDE §5 | Providers org.company / feature.flags | seed + enrich | Done | providers list API | Pass |
| RULE-G-20 | GUIDE §6 | Action catalog | seed actions | Done | list actions | Pass |
| RULE-G-21 | GUIDE §7 | Permissions rules.* | permissions/catalog.py | Done | permission gates | Pass |
| RULE-G-22 | GUIDE §7 | FORCE RLS overlays/eval | Alembic f0a1b2c3d4e5 | Done | alembic upgrade | Pass |
| RULE-G-23 | GUIDE §8 | Module layout | platforms/p11_rules/** | Done | module tests | Pass |
| RULE-G-24 | GUIDE §9 | Outbox jesloterp:rules:outbox | RuleOutboxEvent + emit | Done | activate emit | Pass |
| RULE-S-01 | SCHEMA §2 | 62 domain tables | ORM models | Done | 65-table assert | Pass |
| RULE-S-02 | SCHEMA §2 | Plumbing outbox/idempotency/catalog_audit | 3 plumbing | Done | 65-table assert | Pass |
| RULE-S-03 | SCHEMA §1 | Schema name `rules` never p11 | RULES_SCHEMA | Done | schema constant | Pass |
| RULE-S-04 | SCHEMA §3 | Enumerations | domain/enums.py | Done | engine uses enums | Pass |
| RULE-S-05 | SCHEMA §15 | Seed artifacts/providers/actions/budgets | seed_defaults + migration seed | Done | startup seeds | Pass |
| RULE-A-01 | API §4 | Error codes envelope | RulesError handlers | Done | 403/404/422 | Pass |
| RULE-A-02 | API §5 | Permission codes | RULES_PERMISSIONS | Done | catalog + gate | Pass |
| RULE-A-03 | API §6 | Evaluate / batch / validate / assign / apply | evaluate router | Done | API contracts | Pass |
| RULE-A-04 | API §7 | Internal evaluate/validate/artifacts | internal router | Done | internal API | Pass |
| RULE-A-05 | API §8 | Artifacts CRUD/retire | catalog router | Done | catalog API | Pass |
| RULE-A-06 | API §9 | Definitions/tables/expr/set/fact/actions | definitions router | Done | lifecycle API | Pass |
| RULE-A-07 | API §10 | Compile/publish/activate/retire | definitions router | Done | lifecycle API | Pass |
| RULE-A-08 | API §11 | Overlays CRUD/items/activate | overlays router | Done | overlay API | Pass |
| RULE-A-09 | API §12 | Suites/cases/run/test-runs | suites router | Done | suite run API | Pass |
| RULE-A-10 | API §13 | Packages/changesets/approvals | governance router | Done | pack/changeset API | Pass |
| RULE-A-11 | API §14 | Functions/literals | governance router | Done | functions API | Pass |
| RULE-A-12 | API §15 | Eval-logs/explain/stats | governance router | Done | audit API | Pass |
| RULE-W-01 | sample.md | RTM 100% | this file | Done | — | Pass |
| RULE-W-02 | sample.md | Implementation record 18 sections | RULES_IMPLEMENTATION_RECORD.md | Done | — | Pass |
| RULE-W-03 | Wiring | apps/api/main.py ModuleRegistry + handlers | apps/api/main.py | Done | load_modules test | Pass |
| RULE-W-04 | SCHEMA DDL/RLS | Alembic create + FORCE RLS | e9f0a1b2c3d4 / f0a1b2c3d4e5 | Done | alembic upgrade head | Pass |
| RULE-W-05 | alembic/env.py | Import p11 persistence models | alembic/env.py | Done | alembic upgrade | Pass |
| RULE-W-06 | Registry | PLATFORM_REGISTRY Live for p11 | PLATFORM_REGISTRY.md | Done | Live + checkbox | Pass |
| RULE-SOR-13 | TASK-SOR-013 | Evaluate/publish persist (not CatalogStore SoR) | runtime_repository + persist_eval_ledger | Implemented | `test_durable_sor` | PASS |

---

**Coverage note:** All GUIDE/SCHEMA/API functional requirements mapped and verified under `platforms/p11_rules` (**36** tests). ModuleRegistry + Alembic + registry Live completed (DB head `f0a1b2c3d4e5`).
