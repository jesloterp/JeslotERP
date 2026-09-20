# JeslotERP Rules Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — evaluate persists audit; compile/publish persist compiled artifact; empty list is `[]`. Not Production.  
**Package:** `platforms.p11_rules`  
**PostgreSQL schema:** `rules`  
**Public base:** `/api/v1/rules`  
**Internal base:** `/internal/v1/rules`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`RULES_GUIDE.md`](RULES_GUIDE.md) · [`RULES_SCHEMA.md`](RULES_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Evaluate/explain, decision tables, expressions, rule sets, overlays, simulation, publish, packs, context providers, action suggestions, internal process gateway eval. |
| 1.1 | 2026-09-12 | TASK-SOR-013: evaluate/publish persist; empty catalog/eval-logs is `[]`. |

---

## 1. Design principles (advanced)

1. **Evaluate-first** — production callers use `/evaluate` and `/validate`; authoring is secondary.  
2. **Pure by default** — `/evaluate` does not mutate domain state or auto-fire side effects.  
3. **Explain on demand** — sensitive artifacts may force explain storage.  
4. **Pinable versions** — optional `definition_version` for replay.  
5. **Effective overlays** — tenant/company layers applied server-side.  
6. **Strict facts** when configured — unknown/missing → `FACT_INVALID`.  
7. **Complexity budgets** — timeout and depth errors are first-class.  
8. **Idempotent publish/install** — `Idempotency-Key`.  
9. **Simulation gate** — activate may require green suite.  
10. **Draft evaluate** — separate permission; never default for process runtime.  
11. **Action suggestions only** — unless `/evaluate-apply` with explicit permission.  
12. **Deterministic `as_of`** for time-based rows.

---

## 2. Common headers

```http
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: <uuid>
Idempotency-Key: <key>
X-Tenant-Id: <uuid>
X-Company-Id: <uuid>
```

---

## 3. Envelope

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {
    "request_id": "…",
    "evaluation_id": "…",
    "duration_ms": 4
  }
}
```

---

## 4. Errors

```text
RULE_ARTIFACT_NOT_FOUND / DEFINITION_NOT_FOUND / DEFINITION_NOT_ACTIVE
RULE_FACT_INVALID / FACT_REQUIRED / TYPE_MISMATCH
RULE_AST_DENIED / AST_INVALID / BUDGET_EXCEEDED / EVAL_TIMEOUT
RULE_NO_HIT / MULTI_HIT_ERROR / HIT_POLICY_VIOLATION
RULE_OVERLAY_CONFLICT / OVERLAY_NOT_ACTIVE
RULE_COMPILE_FAILED / PUBLISH_CONFLICT / SIMULATION_FAILED
RULE_APPROVAL_REQUIRED / ACTIVATE_BLOCKED
RULE_PROVIDER_TIMEOUT / PROVIDER_DENIED
RULE_ACTION_UNKNOWN / APPLY_DENIED
RULE_PACKAGE_CHECKSUM_MISMATCH
RULE_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
RULE_DRAFT_EVAL_DENIED
```

HTTP: `404` · `409` · `422` · `403` · `412` · `504` (provider/timeout).

---

## 5. Permissions

| Code | Use |
|---|---|
| `rules.catalog.read` | Read published catalog |
| `rules.catalog.manage` | Edit drafts |
| `rules.evaluate` | Runtime evaluate |
| `rules.evaluate.draft` | Draft dry-run |
| `rules.evaluate.apply` | Evaluate-and-apply actions |
| `rules.publish` | Publish/activate |
| `rules.approve` | Approvals |
| `rules.overlay.manage` | Overlays |
| `rules.simulate` | Suites |
| `rules.pack.install` | Packs |
| `rules.audit.read` | Eval logs |
| `rules.*` | All |

---

## 6. Evaluate APIs (primary runtime)

### 6.1 Evaluate decision / expression / set

```http
POST /api/v1/rules/evaluate
```

```json
{
  "rule_key": "finance.credit.requires_approval",
  "facts": {
    "amount": 150000,
    "currency": "INR",
    "company_id": "…"
  },
  "company_id": "…",
  "as_of": "2026-09-09T04:00:00Z",
  "explain": true,
  "enrich_context": true,
  "mode": "published"
}
```

**Response `data`:**

```json
{
  "evaluation_id": "…",
  "rule_key": "finance.credit.requires_approval",
  "definition_version": 3,
  "status": "OK",
  "result": {
    "requires_finance": true,
    "reason_code": "HIGH_VALUE"
  },
  "hit": {
    "kind": "DECISION_TABLE",
    "row_key": "r_high_enterprise",
    "hit_policy": "FIRST"
  },
  "explain": [
    { "step": "enrich", "provider": "org.company", "added": ["company.type"] },
    { "step": "match", "row_key": "r_high_enterprise", "inputs": { "amount>=": 100000 } }
  ],
  "suggested_actions": [
    { "action_key": "process.edge.select", "params": { "edge": "finance" } }
  ]
}
```

`mode=draft` requires `rules.evaluate.draft`.

### 6.2 Batch evaluate

```http
POST /api/v1/rules/evaluate-batch
```

```json
{
  "requests": [
    { "client_ref": "a", "rule_key": "process.gate.amount_gt_100k", "facts": { "amount": 90000 } },
    { "client_ref": "b", "rule_key": "process.gate.amount_gt_100k", "facts": { "amount": 120000 } }
  ]
}
```

Max batch size from budget policy.

### 6.3 Validate (validation set)

```http
POST /api/v1/rules/validate
```

```json
{
  "ruleset_key": "sales.order.validation",
  "facts": { "weight_kg": -1, "gstin": "" },
  "explain": true
}
```

**Response:**

```json
{
  "ok": false,
  "violations": [
    {
      "rule_key": "bilty.weight.positive",
      "severity": "BLOCKER",
      "message_key": "validation.bilty.weight.positive",
      "path": "weight_kg"
    }
  ]
}
```

`ok=false` if any BLOCKER/ERROR per policy.

### 6.4 Assignment hint

```http
POST /api/v1/rules/assign
```

Returns queue/role/user hints from assignment tables (used by process agent strategies / domain).

### 6.5 Evaluate-and-apply (restricted)

```http
POST /api/v1/rules/evaluate-apply
Idempotency-Key: …
```

Same as evaluate, then executes allow-listed `suggested_actions` via gateways. Requires `rules.evaluate.apply`. Default off for interactive UI.

---

## 7. Internal evaluate (process / domain)

```http
POST /internal/v1/rules/evaluate
POST /internal/v1/rules/validate
GET  /internal/v1/rules/artifacts/{rule_key}
```

Used by p10 XOR `RULE_KEY` and domain pre-save hooks. Pass `tenant_id` explicitly + internal token.

**Process contract:** boolean or `{ "match": true }` depending on edge `condition_value` mapping — documented per artifact; prefer structured result with boolean output field `match`.

---

## 8. Catalog — artifacts

```http
GET    /api/v1/rules/artifacts
POST   /api/v1/rules/artifacts
GET    /api/v1/rules/artifacts/{rule_key}
PATCH  /api/v1/rules/artifacts/{rule_key}
POST   /api/v1/rules/artifacts/{rule_key}/retire
```

---

## 9. Definitions — tables & expressions

### 9.1 Versions

```http
GET    /api/v1/rules/artifacts/{rule_key}/definitions
POST   /api/v1/rules/artifacts/{rule_key}/definitions
GET    /api/v1/rules/definitions/{definition_id}
POST   /api/v1/rules/definitions/{definition_id}/clone
```

### 9.2 Decision table authoring

```http
GET    /api/v1/rules/definitions/{definition_id}/decision-table
PUT    /api/v1/rules/definitions/{definition_id}/decision-table
```

**Put body (draft only):**

```json
{
  "hit_policy": "FIRST",
  "inputs": [
    { "name": "amount", "fact_path": "amount", "data_type": "NUMBER" },
    { "name": "company_type", "fact_path": "company.type", "data_type": "STRING" }
  ],
  "outputs": [
    { "name": "requires_finance", "fact_path": "requires_finance", "data_type": "BOOLEAN" },
    { "name": "reason_code", "fact_path": "reason_code", "data_type": "STRING" }
  ],
  "rows": [
    {
      "row_key": "r_high_enterprise",
      "position": 1,
      "cells": [
        { "input": "amount", "op": "GTE", "value": 100000 },
        { "input": "company_type", "op": "EQ", "value": "ENTERPRISE" },
        { "output": "requires_finance", "value": true },
        { "output": "reason_code", "value": "HIGH_VALUE" }
      ]
    },
    {
      "row_key": "r_default",
      "position": 100,
      "cells": [
        { "input": "amount", "op": "ANY" },
        { "input": "company_type", "op": "ANY" },
        { "output": "requires_finance", "value": false },
        { "output": "reason_code", "value": "DEFAULT" }
      ]
    }
  ]
}
```

### 9.3 Expression authoring

```http
PUT /api/v1/rules/definitions/{definition_id}/expression
POST /api/v1/rules/expressions/validate-ast
```

```json
{
  "return_type": "BOOLEAN",
  "source_text": "amount >= 100000",
  "ast": {
    "type": "gte",
    "left": { "type": "ref", "path": "amount" },
    "right": { "type": "literal", "value": 100000 }
  }
}
```

### 9.4 Rule set / validation set

```http
PUT /api/v1/rules/definitions/{definition_id}/set-members
PUT /api/v1/rules/definitions/{definition_id}/validation
```

### 9.5 Fact schema & actions

```http
PUT /api/v1/rules/definitions/{definition_id}/fact-schema
PUT /api/v1/rules/definitions/{definition_id}/context-bindings
PUT /api/v1/rules/definitions/{definition_id}/action-bindings
GET /api/v1/rules/actions
GET /api/v1/rules/context-providers
```

---

## 10. Compile / publish / activate

```http
POST /api/v1/rules/definitions/{definition_id}/compile
POST /api/v1/rules/definitions/{definition_id}/publish
POST /api/v1/rules/definitions/{definition_id}/activate
POST /api/v1/rules/definitions/{definition_id}/retire
```

**Activate body (optional):**

```json
{
  "require_simulation_green": true,
  "suite_key": "finance.credit.requires_approval.default"
}
```

Fails with `RULE_SIMULATION_FAILED` / `RULE_ACTIVATE_BLOCKED` if gate enabled and red.

---

## 11. Overlays (tenant / company)

```http
GET    /api/v1/rules/overlays?rule_key=…
POST   /api/v1/rules/overlays
GET    /api/v1/rules/overlays/{overlay_id}
PUT    /api/v1/rules/overlays/{overlay_id}/items
POST   /api/v1/rules/overlays/{overlay_id}/activate
POST   /api/v1/rules/overlays/{overlay_id}/deactivate
```

**Item example:**

```json
{
  "op": "ADD_ROW",
  "payload": {
    "row_key": "tenant_special",
    "position": 0,
    "cells": [
      { "input": "amount", "op": "GTE", "value": 75000 },
      { "output": "requires_finance", "value": true },
      { "output": "reason_code", "value": "TENANT_POLICY" }
    ]
  }
}
```

---

## 12. Simulation / test suites

```http
GET    /api/v1/rules/suites
POST   /api/v1/rules/suites
GET    /api/v1/rules/suites/{suite_key}/cases
PUT    /api/v1/rules/suites/{suite_key}/cases
POST   /api/v1/rules/suites/{suite_key}/run
GET    /api/v1/rules/test-runs/{run_id}
```

**Case:**

```json
{
  "name": "high value enterprise",
  "facts": { "amount": 150000, "company": { "type": "ENTERPRISE" } },
  "expected_result": { "requires_finance": true, "reason_code": "HIGH_VALUE" },
  "expected_row_key": "r_high_enterprise",
  "expect_status": "OK"
}
```

---

## 13. Packages & governance

```http
GET  /api/v1/rules/packages
POST /api/v1/rules/packages/{package_key}/install
GET  /api/v1/rules/changesets
POST /api/v1/rules/changesets
POST /api/v1/rules/changesets/{id}/approvals
```

---

## 14. Functions & literals library

```http
GET  /api/v1/rules/functions
POST /api/v1/rules/functions
PUT  /api/v1/rules/functions/{function_key}
GET  /api/v1/rules/literals
PUT  /api/v1/rules/literals/{literal_key}
```

---

## 15. Audit & stats

```http
GET /api/v1/rules/eval-logs?rule_key=…&from=…&to=…
GET /api/v1/rules/eval-logs/{evaluation_id}/explain
GET /api/v1/rules/stats?rule_key=…
```

Requires `rules.audit.read`. Sensitive facts may be hashed only.

---

## 16. Caching

| Resource | Strategy |
|---|---|
| ACTIVE compiled artifact | Memory/Redis by rule_key + overlay fingerprint |
| Invalidate | On activate / overlay activate / pack install |
| Evaluate | No user-specific cache unless facts_hash + version key |

---

## 17. Example client flows

### 17.1 Process XOR gateway

1. p10 edge `condition_type=RULE_KEY`, value `process.gate.amount_gt_100k`  
2. Internal evaluate with instance variables as facts  
3. Boolean `match` chooses edge  

### 17.2 Bilty save validation

1. Domain builds facts from draft bilty  
2. `POST /validate` ruleset `sales.order.validation`  
3. On blockers → 422 to client with message_keys  

### 17.3 Tenant tightens credit threshold

1. Create overlay ADD_ROW at position 0 for 75k  
2. Activate overlay  
3. Evaluate uses effective table; explain shows overlay row  

### 17.4 Author → activate

1. Edit draft decision table  
2. Compile  
3. Run suite  
4. Publish → approve → activate  

---

## 18. Event hooks

| Event | Consumer |
|---|---|
| `rules.definition.activated` | Cache bust (p10/domain) |
| `rules.overlay.changed` | Cache bust tenant |
| `rules.simulation.failed` | Block release pipeline |
| `rules.pack.installed` | Seed notify |

---

## 19. Compatibility notes

- Public prefix `/api/v1/rules`; schema `rules`.  
- Align AST safety with p05 expression policy where practical (shared library later).  
- p10 should not embed thresholds; always RULE_KEY for business cutovers.  
- Message keys resolved via p06 by callers when presenting violations.

---

## 20. Related documents

- Guide: [`RULES_GUIDE.md`](RULES_GUIDE.md)  
- Schema: [`RULES_SCHEMA.md`](RULES_SCHEMA.md)  
- Process: [`../10_process/PROCESS_API.md`](../10_process/PROCESS_API.md)  
- Metadata: [`../05_metadata/METADATA_API.md`](../05_metadata/METADATA_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
