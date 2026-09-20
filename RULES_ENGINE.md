# Rules Engine

**Package:** `p11_rules`  
**Status:** PARTIALLY_IMPLEMENTED  
**Kernel:** SoR-Live (evaluate logs and compiled artifacts persist)  
**Production:** withheld

## Purpose

`p11_rules` is the decision plane. It answers *what is the decision?* Process
answers *who acts next?* Domain modules persist the business effect.

## Current capability

- Rule catalog and definition lifecycle (draft → publish → activate → retire).
- Decision tables and expression definitions.
- Evaluate, batch, and assign APIs with explain traces.
- Tenant / company overlays.
- Simulation suites.
- Eval logs and compiled artifacts on the persistence path.
- Allow-listed AST; no arbitrary code, SQL, or network from evaluate.
- Evaluate is conceptually pure; optional telemetry writes only.

## Planned / incomplete

- Evaluate compute exclusively from compiled artifacts (still mixed with an in-process catalog on part of the path).
- Full DMN product depth (DRD, every hit policy, every FEEL dialect) is **not** claimed.
- End-to-end process gateway `RULE_KEY` wiring is **recommended**, not fully verified.
- Production label.

## Concepts

| Concept | Meaning |
|---|---|
| Rule definition | Versioned decision unit |
| Decision table | Inputs, outputs, rows, hit policy |
| Expression | Allow-listed AST |
| Rule set / agenda | Ordered group |
| Overlay | Tenant or company delta |
| Explain trace | Which row / expression fired |
| Action catalog | Declared side-effect *suggestions*; caller executes |
| Compiled artifact | Published executable form |

## Execution context (public)

Facts are a JSON payload plus scoped context (tenant, company, actor). Facts
may map to metadata fields. The engine does not reach into another schema.

## Priorities and lifecycle

Overlays resolve SYSTEM → PACK → TENANT → COMPANY. Runtime pins the ACTIVE
definition. Simulation should pass before activate.

## Security

- `rules.*` permissions, including a distinct apply / evaluate split.
- RLS on access.
- AST allow-list and evaluation budgets.
- No stored Python execution.

## Event-driven execution

Definitions emit publish / activate / overlay / simulation events. Callers
(process, domain validate, future pricing) invoke evaluate; the engine does
not subscribe itself as a general-purpose job worker.

## Future configuration-driven business logic

Intended: credit checks, tax applicability hints, approval thresholds, and
output determination predicates become published rules. **Not implemented** as
business content today.

This document does not invent request/response schemas beyond the resource
groups listed in [PACKAGES/P11_RULES.md](PACKAGES/P11_RULES.md).
