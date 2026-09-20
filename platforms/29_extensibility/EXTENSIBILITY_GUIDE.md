# JeslotERP Extensibility Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — Alembic `f29a0b1c2d3e` / `f29b1c2d3e4f`; not Production  
**Package:** `platforms.p29_extensibility`  
**PostgreSQL schema:** `extensibility`  
**Depends on:** `p01_identity`  
**Integrates with:** `p05_metadata` (points on entities), `p11_rules` (validation facts), `p10_process` (hooks), `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`EXTENSIBILITY_SCHEMA.md`](EXTENSIBILITY_SCHEMA.md) · [`EXTENSIBILITY_API.md`](EXTENSIBILITY_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | Extension points, allow-listed handlers, before/after/validate/compensate pipeline. AUD-005. |

---

## 1. Purpose (enterprise)

`p29_extensibility` is JeslotERP’s **enhancement / plugin / hook runtime** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP BAdI / enhancement spots / user-exits**  
- **Dynamics 365 plugin pipeline** (pre-validation / pre-operation / post-operation)  
- **Salesforce Apex triggers / Flow triggers** (before/after)  

It is **not** arbitrary Python from the database. It is a **safe allow-listed execution model**.

### Owns

| Domain | Examples |
|---|---|
| Extension points | Named hook slots (`bp.partner.before_save`) |
| Handlers | Allow-listed handler keys + module binding |
| Bindings | Point + handler + priority + activation |
| Pipeline | BEFORE / VALIDATE / AFTER / COMPENSATE |
| Execution log | Status, duration, failure policy |
| Allow-list | Handler keys permitted to run |
| Failure policy | FAIL_CLOSED / CONTINUE / COMPENSATE |

### Does **not** own

| Concern | Owner |
|---|---|
| Field dictionary / custom fields | `p05_metadata` |
| Decision tables / AST evaluate | `p11_rules` |
| Workflow inbox | `p10_process` |
| CRM / sales trigger logic | `business/bNN_*` (future consumers) |
| Arbitrary script hosting | **Nobody** — denied |

### Critical split: Metadata vs Rules vs Extensibility

| | **p05 Metadata** | **p11 Rules** | **p29 Extensibility** |
|---|---|---|---|
| Question | What is the shape? | What is the decision? | What extra work runs around an operation? |
| Output | Effective field/UI | `{ result, explain }` | Handler results + compensation |
| Safety | AST for formulas | Allow-listed AST | Allow-listed handler_key only |

**Rule:** Callers invoke `execute(point_key, phase, context)`. p29 never `eval()` / `exec()` / `import payload["module"]`.

---

## 2. Architectural position

```text
Caller (p04/p05/p10/future bNN)
        │
        ▼
  execute(point_key, phase, context)
        │
        ▼
  Binding resolver (ACTIVE, priority, tenant overlay)
        │
        ▼
  Allow-list + timeout + failure policy
        │
        ▼
  Handler port (registered in-process implementations)
        │
        ├── success → AFTER
        └── failure → FAIL_CLOSED | CONTINUE | COMPENSATE
```

**Hard rules**

1. Handlers are allow-listed (`handler_key`).  
2. No `eval`, `exec`, unrestricted Python.  
3. PostgreSQL SoR for points/bindings/executions.  
4. No cross-schema FKs.  
5. Generic resource ids — no sales-order types.

---

## 3. Advanced design principles

1. **Point is the contract** — stable `point_key` + phase + context schema.  
2. **Priority order** — lower number first (enhancement-point order).  
3. **Activation** — DRAFT → ACTIVE → INACTIVE.  
4. **Timeouts** — per binding; default fail-closed. Overtime kills the isolated worker process (`terminate` then `kill`).  
5. **Idempotent bind/execute** — `Idempotency-Key`.  
6. **Tenant overlay** — tenant bindings add/disable system bindings.  
7. **Audit** — every execute writes `ext_execution`.  
8. **Compensation** — COMPENSATE phase on FAIL_CLOSED after AFTER started.  
9. **Version** on handler + binding.  
10. **No business modules inside p29.**

---

## 4. Core concepts

### 4.1 Phases

| Phase | When |
|---|---|
| `BEFORE` | Before the operation mutates |
| `VALIDATE` | Validation-only; no writes |
| `AFTER` | After success |
| `COMPENSATE` | Undo AFTER work on later failure |

### 4.2 Handler registry

```text
handler_key = "ext.sample.noop"
kind = BUILTIN
timeout_ms, is_active
```

Only `BUILTIN` handlers registered in application code are executable in v1.

### 4.3 Execution context

```text
point_key, phase, tenant_id, actor_id,
resource_type, resource_id,
payload (JSON), correlation_id
```

### 4.4 Failure policy

| Policy | Behavior |
|---|---|
| `FAIL_CLOSED` | Abort caller operation |
| `CONTINUE` | Log and proceed |
| `COMPENSATE` | Run COMPENSATE handlers |

---

## 5. Security

### Permissions

| Code | Use |
|---|---|
| `extensibility.point.read` | List points |
| `extensibility.point.manage` | Create points |
| `extensibility.handler.manage` | Register metadata |
| `extensibility.bind` | Bind/unbind |
| `extensibility.execute` | Run pipeline |
| `extensibility.admin` | Allow-list |
| `extensibility.audit.read` | Executions |
| `extensibility.*` | Wildcard |

### RLS

FORCE RLS on tenant-scoped bindings and executions.

---

## 6. Module layout

```text
platforms/p29_extensibility/
  application/services/extensibility_service.py
  application/hooks.py    # invoke_hooks: p04 save/delete/activate/blacklist, p05 publish, p10 start/cancel/signal/complete_task
  application/handlers/   # builtin only
  application/ports/handler.py
  infrastructure/http/… persistence/…
```

---

## 7. Domain events

| Event | When |
|---|---|
| `extensibility.binding.activated` | Bind ACTIVE |
| `extensibility.execution.failed` | FAIL_CLOSED |
| `extensibility.point.created` | Catalog |

Stream: `jesloterp:extensibility:outbox`.

---

## 8. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs |
| P1 | Points, handlers, allow-list, RLS |
| P2 | Bind + activate/deactivate |
| P3 | Execute pipeline + timeout |
| P4 | Failure policy + compensate |
| P5 | Tests + **SoR-Live** |

---

## 9. Definition of Done (enterprise)

- [x] Unknown handler_key rejected  
- [x] Inactive binding skipped  
- [x] Priority order unit-tested  
- [x] Timeout enforced  
- [x] P29-LIVE-001: timeout kills isolated worker process (`EXT_TIMEOUT`; child `is_alive()` is False)  
- [x] P29-LIVE-002: consumers beyond create/publish/start — p04 update/delete, p05 after_publish, p10 cancel  
- [x] FAIL_CLOSED stops subsequent handlers  
- [x] No eval/exec in package  
- [x] Tenant RLS on executions  
- [x] K28-33-001: list bindings/executions DB-first on `AsyncSession` (empty is `[]`)  
- [x] No CRM-specific handlers  

---

## 10. Anti-patterns

| Don’t | Do |
|---|---|
| `eval(payload["code"])` | Allow-listed handler_key |
| Sales-order trigger in p29 | Generic `resource_type` |
| Silent handler failure | Failure policy + log |
| HTTP → ORM | HTTP → PipelineService |

---

## 11. Related documents

- Schema: [`EXTENSIBILITY_SCHEMA.md`](EXTENSIBILITY_SCHEMA.md)  
- API: [`EXTENSIBILITY_API.md`](EXTENSIBILITY_API.md)  
- RTM: [`EXTENSIBILITY_RTM.md`](EXTENSIBILITY_RTM.md)  
- Implementation record: [`EXTENSIBILITY_IMPLEMENTATION_RECORD.md`](EXTENSIBILITY_IMPLEMENTATION_RECORD.md)  
- Rules AST bar: [`../11_rules/RULES_GUIDE.md`](../11_rules/RULES_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
