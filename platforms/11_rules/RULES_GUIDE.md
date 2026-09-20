# JeslotERP Rules Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — evaluate writes `rule_eval_log`; publish/compile writes `rule_compiled_artifact`. Empty catalog is `[]`. Not Production.  
**Package:** `platforms.p11_rules`  
**PostgreSQL schema:** `rules`  
**Depends on:** `p03_configuration`, `p05_metadata` (registry)  
**Integrates with:** `p01_identity` (authz), `p02_organization` (scope), `p06_localization` (messages), `p10_process` (gateway `RULE_KEY`), `p12_feature`, `p16_cache`, `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`RULES_SCHEMA.md`](RULES_SCHEMA.md) · [`RULES_API.md`](RULES_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise decision plane: DMN-class tables, hit policies, AST expressions, rule sets, agendas, explain traces, simulation, tenant overlays, actions catalog, compile cache, packs, governance. |
| 1.1 | 2026-09-12 | TASK-SOR-013: evaluate/publish persist; empty list is `[]`; RLS on `require_rules_access`. |

---

## 1. Purpose (enterprise)

`p11_rules` is JeslotERP’s **business rules & decision control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP BRF+ / Decision Service Management** — functions, decision tables, catalog, versioning  
- **Microsoft Dynamics 365** — business rules, decision tables, calculated logic without code deploys  
- **Salesforce** — validation rules, assignment rules, orchestration decisions  
- **DMN 1.x** — decision tables, hit policies, literal expressions, decision requirements  

It is **not** scattered `if amount > 100000` in Python services. It is the system that makes ERP decisions:

1. **Deterministic & explainable** — which rule/row fired and why  
2. **Versioned & publishable** — runtime pins ACTIVE definition  
3. **Safe** — allow-listed AST; no arbitrary code, SQL, or network  
4. **Scoped** — system → pack → tenant → company overlays  
5. **Reusable** — rule sets invoked by process gateways, validators, pricing hints, credit checks  
6. **Testable** — simulation suites & golden cases before activate  
7. **Side-effect controlled** — pure evaluate by default; explicit action bindings for domain hooks  

### Owns

| Domain | Examples |
|---|---|
| Rule catalog | rule_key, categories, owners |
| Decision tables | Inputs/outputs/rows, hit policy |
| Expressions | AST literal expressions |
| Rule sets / agendas | Ordered evaluation groups |
| Decision graphs | DRD-lite dependencies |
| Overlays | Tenant/company row overlays |
| Evaluation runtime | Evaluate + explain trace |
| Simulation / tests | Cases, expected hits |
| Action catalog | Declared side-effect hooks (invoked by callers) |
| Compile cache | Compiled table/expr artifacts |
| Governance | Publish, packs, approvals |

### Does **not** own

| Concern | Owner |
|---|---|
| Workflow tokens / inbox | `p10_process` |
| Setting values | `p03_configuration` |
| Entity field dictionary | `p05_metadata` (facts may map to fields) |
| Actual credit posting / bilty mutate | Domain modules |
| ML model scoring | `p27_ai` (may feed facts; not rules engine) |
| Feature flag storage | `p12_feature` (fact provider) |

### Critical split: Rules vs Process vs Domain

| | **Rules (p11)** | **Process (p10)** | **Domain** |
|---|---|---|---|
| Question | What is the decision? | Who does the next human/system step? | Persist business effect |
| Output | `{ result, hit, explain }` | Work items / tokens | Updated bilty/invoice |
| Purity | Evaluate is pure | Orchestration | Mutations |

**Rule:** Process XOR edges call `RULE_KEY` → p11 evaluate → choose path. Domain may also call evaluate for validation before save.

---

## 2. Architectural position

```text
Facts (payload + context providers)
         │
         ▼
   Effective rules resolver
   SYSTEM → PACK → TENANT → COMPANY
         │
         ▼
   Decision engine (table / expression / ruleset)
         │
         ├── result + explain trace
         └── (optional) action suggestions — caller executes
```

**Hard rules**

1. Evaluate has **no DB writes** except optional audit/telemetry.  
2. No cross-schema FKs — UUID refs; facts are JSON.  
3. Expressions use **allow-listed AST** (same safety bar as p05).  
4. ACTIVE definitions immutable; edit via new version.  
5. RLS fail-closed on tenant overlays & eval audit if stored.

---

## 3. Advanced design principles

1. **Decision-first** — prefer decision tables for business-editable logic.  
2. **Hit policies** — FIRST, UNIQUE, ANY, PRIORITY, COLLECT (with aggregators).  
3. **Explainability mandatory** for audit-sensitive decisions.  
4. **Fact schema** — declared inputs with types; unknown facts rejected in strict mode.  
5. **Context providers** — optional enrichment (org, feature flags) via allow-listed providers.  
6. **Layered effective resolve** — overlays add/replace rows by priority.  
7. **Compile on publish** — validate + compile artifact + checksum.  
8. **Simulation gate** — required suite green before activate (policy).  
9. **Deterministic time** — `as_of` / `evaluation_time` injected; no wall-clock drift in tests.  
10. **Localization** — message keys for validation failures, not hardcoded English.  
11. **Idempotent evaluate** — same facts + version → same result.  
12. **Action catalog ≠ auto-execute** — engine returns `suggested_actions`; domain/process executes.  
13. **Complexity budgets** — max rows, max AST depth, max eval ms.  
14. **CQRS HTTP** — thin routers; engine in application services.  
15. **Cache** — compiled defs by `(rule_key, version, tenant, company)`.  
16. **Dry-run** — evaluate against DRAFT with manage permission.  
17. **Decision requirements** — decision A may require B’s output as input.  
18. **Packages** — India GST validation pack, credit matrix pack.

---

## 4. Core concepts

### 4.1 Rule artifact kinds

| Kind | Use |
|---|---|
| `DECISION_TABLE` | Primary business matrix |
| `EXPRESSION` | Single AST expression → value/bool |
| `RULE_SET` | Ordered list of rules/decisions |
| `DECISION_CHAIN` | DRD-lite: evaluate dependencies first |
| `VALIDATION_SET` | Collect all failed validations |
| `ASSIGNMENT_TABLE` | Map facts → assignee/queue hints |
| `SCORECARD` | Weighted scoring (optional phase) |

### 4.2 Decision table anatomy

```text
Inputs:  amount:NUMBER, company_type:STRING, lane:STRING
Outputs: requires_finance:BOOLEAN, reason_code:STRING
Hit:     FIRST
Rows:
  amount >= 100000 AND company_type = "ENTERPRISE" → true, "HIGH_VALUE"
  amount >= 50000 → true, "MID_VALUE"
  - → false, "DEFAULT"
```

### 4.3 Hit policies

| Policy | Behavior |
|---|---|
| `FIRST` | First matching row (ordered) |
| `UNIQUE` | Exactly one match or error |
| `ANY` | All matches must agree outputs |
| `PRIORITY` | Highest priority matching row |
| `COLLECT` | All matches; aggregate (list/sum/min/max/count) |

### 4.4 Expression AST (allow-list)

Supported nodes (illustrative):

```text
literal, ref(fact), path,
eq/neq/gt/gte/lt/lte,
and/or/not,
in/contains,
coalesce, if,
add/sub/mul/div,
round, abs,
date add/diff (bounded),
string lower/upper/len,
list len / includes
```

Denied: loops unbounded, network, SQL, import, eval, mutation, regex catastrophic backtracking (or tightly limited).

### 4.5 Effective layers

| Priority | Layer |
|---:|---|
| 10 | SYSTEM |
| 20 | PACKAGE |
| 30 | TENANT overlay |
| 40 | COMPANY overlay |

Overlays can `ADD_ROW`, `DISABLE_ROW`, `REPLACE_ROW`, `REPLACE_EXPR`.

### 4.6 Evaluate I/O

**Input:**

```text
rule_key | ruleset_key,
facts{},
tenant_id, company_id?,
as_of?,
definition_version?,   # pin optional
explain?: bool,
mode: published | draft
```

**Output:**

```text
ok, result{},
hit: { artifact_id, row_id?, policy },
explain: [ steps… ],
suggested_actions: [ { action_key, params } ],
evaluation_id, duration_ms, definition_version
```

### 4.7 Validation sets

Evaluate all rules; return `violations[]` with `message_key`, `severity`, `path`.  
Used for “save bilty” pre-checks.

---

## 5. Context providers

Allow-listed providers enrich facts before eval:

| Provider | Facts added |
|---|---|
| `org.company` | company_type, country, gst_status |
| `feature.flags` | flag map |
| `identity.roles` | role codes (careful — authz still separate) |
| `config.settings` | selected non-secret settings |
| `process.variables` | when called from p10 |

Providers have timeouts and field allow-lists.

---

## 6. Actions catalog

Declared actions (not auto-run by default):

```text
notify.template
process.start
document.require_release_process
domain.credit.block
domain.validation.reject
```

Evaluate may return `suggested_actions`; **caller** executes via its gateway.  
Optional `evaluate-and-apply` internal API only for trusted automation with explicit permission.

---

## 7. Security

### Permissions

| Code | Use |
|---|---|
| `rules.catalog.read` | Read published |
| `rules.catalog.manage` | Edit drafts |
| `rules.evaluate` | Runtime evaluate |
| `rules.evaluate.draft` | Dry-run drafts |
| `rules.publish` | Publish/activate |
| `rules.approve` | Approvals |
| `rules.overlay.manage` | Tenant/company overlays |
| `rules.simulate` | Test suites |
| `rules.pack.install` | Packs |
| `rules.audit.read` | Eval/audit logs |
| `rules.*` | Wildcard |

### RLS

FORCE RLS on tenant overlays, simulation tenants, eval audit rows.

---

## 8. Module layout

```text
platforms/p11_rules/
  application/
    services/
      effective_resolver.py
      table_engine.py
      expression_engine.py
      ruleset_engine.py
      explain.py
      compiler.py
      simulator.py
      context_providers.py
      overlay_merger.py
    commands/… queries/…
    permissions/catalog.py
  domain/ast/ …
  infrastructure/
    http/… persistence/… messaging/outbox/
    cache/compiled_store.py
  tests/unit/table/ expr/ hit_policy/ overlay/
```

---

## 9. Domain events

| Event | When |
|---|---|
| `rules.definition.published` / `activated` | Governance |
| `rules.overlay.changed` | Overlay |
| `rules.pack.installed` | Packs |
| `rules.simulation.failed` | Gate fail |
| `rules.evaluate.recorded` | Optional sampled telemetry |

Stream: `jesloterp:rules:outbox`.

---

## 10. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, catalog, permissions |
| P2 | Expression AST + evaluate |
| P3 | Decision tables FIRST/UNIQUE |
| P4 | PRIORITY/COLLECT + explain |
| P5 | Rule sets + validation sets |
| P6 | Overlays + effective resolve |
| P7 | Compile cache + simulation suites |
| P8 | Context providers + action catalog |
| P9 | Packs (credit, bilty validation, process gates) |
| P10 | Registry → **Live** |

---

## 11. Definition of Done (enterprise)

- [x] Golden tests for all hit policies  
- [x] AST deny list blocks unsafe nodes  
- [x] Explain shows matched row ids  
- [x] Overlay precedence unit-tested  
- [x] Publish immutability + checksum  
- [x] Simulation gate can block activate  
- [x] Eval timeout enforced  
- [x] Tenant RLS GUCs on evaluate/audit HTTP (`require_rules_access`)  
- [x] Evaluate logs + compiled artifacts persist on `AsyncSession` (empty catalog is `[]`)  
- [x] No cross-schema FKs  

---

## 12. Anti-patterns

| Don’t | Do |
|---|---|
| Embed business thresholds in code | Decision table rows |
| `eval()` Python from DB strings | Allow-listed AST |
| Silent change of ACTIVE table | Version + activate |
| Engine posts financial documents | Return result; domain posts |
| Unbounded loops in expressions | Complexity budget |
| Skip explain on credit decisions | `explain=true` for sensitive keys |

---

## 13. Related documents

- Schema: [`RULES_SCHEMA.md`](RULES_SCHEMA.md)  
- API: [`RULES_API.md`](RULES_API.md)  
- Process: [`../10_process/PROCESS_GUIDE.md`](../10_process/PROCESS_GUIDE.md)  
- Metadata AST: [`../05_metadata/METADATA_GUIDE.md`](../05_metadata/METADATA_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
