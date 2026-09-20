# JeslotERP Rules Platform (p11) — Implementation Record

**Date:** 2026-09-12  
**Package:** `platforms.p11_rules`  
**PostgreSQL schema:** `rules`  
**Source of truth reviewed:** `RULES_GUIDE.md`, `RULES_SCHEMA.md`, `RULES_API.md`, `docs/sample.md`

---

## 1. Overview & Objective

Deliver the backend business-rules / decision control plane: decision tables with hit policies, allow-listed AST expressions, validation/assignment sets, tenant overlays, compile→publish→activate lifecycle, simulation gate, pure evaluate (+ gated evaluate-apply), permissions (`rules.*`), **65** ORM tables on schema `rules`, HTTP `/api/v1/rules` + `/internal/v1/rules`, ModuleRegistry wiring, Alembic DDL + FORCE RLS.

---

## 2. All 3 Source Documents Reviewed

| Document | Version | Review outcome |
| --- | --- | --- |
| `RULES_GUIDE.md` | 1.0 | Enterprise decision plane — mapped end-to-end |
| `RULES_SCHEMA.md` | 1.0 | 62+3 tables — ORM + Alembic `e9f0a1b2c3d4` / RLS `f0a1b2c3d4e5` |
| `RULES_API.md` | 1.0 | Public + internal routes §§6–15 — contract-tested |

---

## 3. Existing Backend Architecture Reviewed

Mirrored `p10_process` / `p09_document`: ModulePlugin, RulePlatform/Tenant/Catalog bases, RulesCatalogStore in-memory engine, exception handlers, outbox stream `jesloterp:rules:outbox`, permission catalog seed pattern, TestClient auth overrides.

---

## 4. Requirements Identified

Full RTM in `RULES_RTM.md` (GUIDE architecture + SCHEMA inventory + API surface + sample.md RTM/record/wiring).

---

## 5. Requirement-by-Requirement Implementation

| Area | Implementation | Verification |
| --- | --- | --- |
| Decision tables + hit policies | `table_engine.py` | unit hit policy tests |
| AST allow-list + deny | `expression_engine.py` | AST unit tests |
| Evaluate / validate / assign / batch / apply | RulesCatalogStore + evaluate router | API + store tests |
| Overlays ADD/DISABLE/REPLACE | overlay_merger + store | overlay unit + API |
| Compile / publish / activate + sim gate | store lifecycle | SimulationFailedError + API |
| Seeds (credit, gate, bilty validation) | `seed_defaults` | evaluate/validate green |
| Module wiring | apps/api/main.py | test_rules_load_modules_includes_rules |
| DDL + RLS | Alembic e9f0… / f0a1… | upgrade head applied |

---

## 6. Files/Modules/Services Created or Modified

**Platform (new):** `platforms/p11_rules/**`  
- domain: enums, exceptions  
- application: permissions, errors, expression/table/overlay engines, catalog_store  
- infrastructure: ORM (65 tables), HTTP routers, module, outbox, cache stub  
- tests: `test_rules_*` under unit/{api,ast,engine,module,overlay,permissions}

**API wiring:** `RulesModule` + `register_rules_exception_handlers` in `apps/api/main.py`  
**Alembic:** env import; `e9f0a1b2c3d4_create_rules_schema.py`; `f0a1b2c3d4e5_enable_rules_rls.py`  
**Docs:** RTM, implementation record; GUIDE/SCHEMA/API Status → Live; registry Live

---

## 7. Database Changes & Migrations

| Revision | Purpose | Applied |
| --- | --- | --- |
| `e9f0a1b2c3d4` | CREATE SCHEMA `rules`; create_all 65 tables; seed AST allow, budgets, providers, actions; seed `rules.*` permissions + admin grants | Yes (`alembic upgrade head`) |
| `f0a1b2c3d4e5` | ENABLE + FORCE RLS on tenant-scoped overlay/eval/test/changeset tables | Yes |

Down-revision chain: `d8e9f0a1b2c3` → `e9f0a1b2c3d4` → `f0a1b2c3d4e5` (head).

---

## 8. APIs/Endpoints Implemented or Updated

**Public `/api/v1/rules`:** evaluate, evaluate-batch, validate, assign, evaluate-apply; artifacts CRUD/retire; definitions CRUD/clone; decision-table GET/PUT; expression PUT + validate-ast; set-members/validation/fact-schema/context/action bindings; actions & providers list; compile/publish/activate/retire; overlays; suites/cases/run/test-runs; packages install; changesets/approvals; functions/literals; eval-logs/explain/stats.

**Internal `/internal/v1/rules`:** POST evaluate, POST validate, GET artifacts/{rule_key}.

---

## 9. Business Rules & Workflows Implemented

- Hit policies FIRST / UNIQUE / ANY / PRIORITY / COLLECT (+ aggregators)  
- Cell ops ANY, EQ, NEQ, GT/GTE/LT/LTE, IN/NOT_IN, BETWEEN, IS_NULL, EXPR  
- Effective overlay merge before evaluate  
- Validation set collects violations with severity; BLOCKER/ERROR → ok=false  
- Assignment table → queue/role hints  
- ACTIVE immutable; simulation gate blocks activate when red  
- Evaluate-apply executes suggested_actions only with `rules.evaluate.apply`

---

## 10. Validation, Permissions & Error Handling

- Error codes per API §4 via `RulesError` + `register_rules_exception_handlers`  
- Permissions per API §5 (`rules.*`, catalog, evaluate, draft, apply, publish, approve, overlay, simulate, pack, audit)  
- Draft evaluate denied without `rules.evaluate.draft`  
- Strict facts → FACT_INVALID / FACT_REQUIRED / TYPE_MISMATCH  
- AST denied → RULE_AST_DENIED

---

## 11. Integrations Implemented

- Soft integrate with process via internal evaluate (RULE_KEY gateway contract: boolean `match` on `process.gate.amount_gt_100k`)  
- Depends on identity (authz), organization (scope), configuration, metadata (ModulePlugin deps)  
- Outbox topic `jesloterp:rules:outbox` for published/activated/overlay/pack/simulation events

---

## 12. Test Cases Created for Each Functionality

Focused files (unique `test_rules_*` prefixes):

| File | Coverage |
| --- | --- |
| `test_rules_hit_policies.py` | All hit policies + cell ops (≥2 each area) |
| `test_rules_ast_engine.py` | Eval success, deny, invalid, budget, if/coalesce |
| `test_rules_overlay_merge.py` | ADD/DISABLE/REPLACE + store evaluate effect |
| `test_rules_store_runtime.py` | Evaluate/validate/strict/draft/sim gate/idempotent/batch/apply |
| `test_rules_api_contracts.py` | Public + internal HTTP success/failure/auth |
| `test_rules_permissions_gate.py` | Catalog codes + wildcard + HTTP 403 |
| `test_rules_module_tables.py` | 65 tables, plugin deps, load_modules |

---

## 13. Test Execution Results

```
pytest platforms/p11_rules/tests -q --tb=line
→ 36 passed
```

Module load: `load_modules()` includes `p11_rules`; `RulesModule().on_startup()` prints initialized.

---

## 14. Requirements Traceability Matrix (RTM)

See `RULES_RTM.md` — 100% mapped GUIDE/SCHEMA/API/wiring requirements with Pass status.

---

## 15. Issues Found & How They Were Resolved

| Issue | Resolution |
| --- | --- |
| PowerShell treats alembic INFO on stderr as error | Used `$ErrorActionPreference='Continue'`; upgrade still applied |
| Overlay test UUID serialization | Normalize via `UUID(str(ov["id"]))` |
| Need unique pytest names vs other platforms | Used `test_rules_*` prefixes only |

---

## 16. Regression/Existing Functionality Verification

- p11 tests isolated under `platforms/p11_rules/tests`  
- ModuleRegistry topo order keeps p10 before p11  
- Existing process migrations remain head parent (`d8e9f0a1b2c3`)

---

## 17. Final Coverage & Completion Status

| Deliverable | Status |
| --- | --- |
| Package + engines + HTTP | Complete |
| 65 ORM tables | Complete (asserted) |
| Alembic DDL + RLS | Applied to head `f0a1b2c3d4e5` |
| ModuleRegistry + exception handlers | Wired |
| RTM + Implementation Record | Complete |
| Registry Live + phase checkbox | Updated |
| GUIDE/SCHEMA/API Status Live | Updated |
| Tests ≥2 variations per major area | 36 passed |

---

## 18. Remaining Issues or Limitations

1. HTTP evaluate writes `rule_eval_log` (+ explain snapshot when present); compile/publish writes `rule_compiled_artifact`. Empty catalog/eval-logs is `[]`. `RulesCatalogStore` remains the evaluation engine / TestClient double. Suites/stats still memory. Not Production.  
2. **Scorecard / DECISION_CHAIN:** Schema + ORM present; evaluate path focuses on DECISION_TABLE, EXPRESSION, VALIDATION_SET, ASSIGNMENT_TABLE seeds (SCORECARD is optional-phase in GUIDE).  
3. **Provider timeouts:** Providers are stub enrichers; RULE_PROVIDER_TIMEOUT path exists as exception type but stubs do not sleep/timeout.  
4. **Eval timeout:** Enforced via max_eval_ms check after evaluation; not a hard wall-clock interrupt mid-AST.  
5. **Postgres repositories / Redis compile cache:** CompiledStore is an in-process stub; production Redis wiring deferred to p16_cache integration.
