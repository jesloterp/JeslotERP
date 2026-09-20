# JeslotERP Process Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — HTTP start/list/get instances + inbox persist/fetch on `AsyncSession`. Not Production.  
**Package:** `platforms.p10_process`  
**PostgreSQL schema:** `process`  
**Depends on:** `p01_identity`, `p02_organization`, `p03_configuration`, `p04_business_partner`, `p09_document` (registry)  
**Integrates with:** `p05_metadata`, `p06_localization`, `p07_number_series`, `p08_file_media`, `p11_rules` (conditions), `p12_feature`, `p14_messaging`, `p15_notification`, `p17_scheduler` (timers), `p19_audit`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`PROCESS_SCHEMA.md`](PROCESS_SCHEMA.md) · [`PROCESS_API.md`](PROCESS_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise BPM/approval plane: definition versions, BPMN-lite graph, work items, agent resolution, SLA/escalation, delegation, parallel/exclusive gateways, subprocess, timers, inbox, correlation, compensation, packs. |
| 1.1 | 2026-09-12 | TASK-SOR-012: HTTP instances + inbox Postgres-first; RLS on `require_process_access`. |

---

## 1. Purpose (enterprise)

`p10_process` is JeslotERP’s **workflow & approval control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Business Workflow / Flexible Workflow** — work items, agent determination, deadlines, events  
- **Microsoft Dynamics 365** — business process flows, approval workflows, queues  
- **Salesforce** — Approval Processes, Flow orchestration, recall/reassign  
- **BPMN 2.0 subset** — user tasks, service tasks, gateways, timers, subprocesses  

It is **not** a boolean `approved` column. It is the system that makes ERP human and system workflows correct for:

1. **Multi-step approvals** (credit override, rate change, bilty cancel, document release)  
2. **Versioned process definitions** with publish/activate  
3. **Work item inbox** — claim, complete, reject, reassign, delegate  
4. **Agent resolution** — user, role, group, manager, org path, BP contact, custom strategy  
5. **Exclusive / parallel gateways** and join synchronization  
6. **SLA timers & escalation** (remind → escalate → auto-action policy)  
7. **Delegation / out-of-office** coverage  
8. **Business correlation** — one instance per (process_key, business_key) when unique  
9. **Subprocesses & call activities**  
10. **Compensation / cancel** paths, audit-grade history, notification hooks  

### Owns

| Domain | Examples |
|---|---|
| Process catalog | Keys, categories, owners |
| Definitions & graphs | Nodes, edges, versions |
| Instances | Running / completed cases |
| Tokens / execution | Where the flow is |
| Work items / tasks | Human inbox items |
| Agent rules | Who should act |
| SLA / deadlines | Timers, escalations |
| Delegation | OOO & substitute |
| Forms / payloads | Decision data schemas |
| Correlation | Business key binding |
| Inbox views | My tasks / team queues |
| Governance | Publish packs, approvals on defs |

### Does **not** own

| Concern | Owner |
|---|---|
| Decision tables / complex rules DSL | `p11_rules` (invoked for conditions) |
| Document content / versions | `p09_document` |
| Party master | `p04_business_partner` |
| Cron infrastructure | `p17_scheduler` (fires timer due jobs) |
| Email/push delivery | `p15_notification` |
| Domain business mutate logic | Domain modules (service tasks call gateways) |
| UI metadata for forms | `p05_metadata` (form keys) |

### Critical split: Process vs Rules vs Domain

| | **Process (p10)** | **Rules (p11)** | **Domain** |
|---|---|---|---|
| Question | What step next? Who acts? | Does condition X hold? | Apply business effect |
| Stores | Instances, tasks, tokens | Rule sets, decision tables | Bilty/invoice state |
| On approve | Complete work item → advance | Evaluate amount>limit | Post cancel / unlock credit |

---

## 2. Architectural position

```text
Domain event / API start
         │
         ▼
  p10 process engine
    ├─ user task ──► inbox ──► notification
    ├─ service task ──► domain gateway
    ├─ exclusive gateway ──► p11 rules (optional)
    ├─ timer ──► scheduler / worker
    └─ end ──► outbox domain callback
```

**Hard rules**

1. Domain **starts** processes and **reacts** to outcomes — engine does not own bilty tables.  
2. No cross-schema FKs — UUID + business_key correlation.  
3. Definition changes never mutate running instances mid-graph without migration policy.  
4. Completing a task is idempotent.  
5. RLS fail-closed on tenant instances & work items.

---

## 3. Advanced design principles

1. **Definition versioning** — DRAFT → PUBLISHED → ACTIVE; instances pin `definition_version_id`.  
2. **BPMN-lite graph** — NODE + EDGE model (not full BPMN XML required v1; import later).  
3. **Token execution** — explicit tokens for parallel paths.  
4. **Work item state machine** — CREATED → READY → RESERVED → COMPLETED \| REJECTED \| CANCELLED \| EXPIRED.  
5. **Claim vs push** — queue claim or direct assignee.  
6. **Agent determination strategies** — pluggable resolvers.  
7. **SLA clock** — from READY or RESERVED per policy.  
8. **Escalation ladder** — N levels with actions.  
9. **Delegation calendar** — substitute receives work.  
10. **Correlation uniqueness** — optional single active instance per business key.  
11. **Variables / payload** — JSONB process variables with schema allow-list.  
12. **Service tasks** — registered handlers only (no arbitrary code).  
13. **Signals / messages** — correlate external events into waiting instances.  
14. **Compensation** — cancel path / undo handlers.  
15. **Ad-hoc tasks** — optional side tasks without leaving main flow.  
16. **Audit everything** — transitions, assignees, comments.  
17. **CQRS HTTP** — engine services isolated.  
18. **Idempotent start/complete/signal**.

---

## 4. Core concepts

### 4.1 Process definition

```text
process_key = "sales.order.cancel.approval"
versions: 1 (ACTIVE), 2 (DRAFT)
graph: start → userTask(manager) → xor(amount?) → userTask(finance) → serviceTask(apply) → end
```

### 4.2 Node kinds (v1)

| Kind | Role |
|---|---|
| `START` | Entry |
| `END` | Terminal success |
| `END_TERMINATE` | Kill parallels |
| `USER_TASK` | Human work item |
| `SERVICE_TASK` | System handler |
| `SCRIPT_TASK` | Restricted expression (optional; prefer service) |
| `EXCLUSIVE_GATEWAY` | XOR |
| `PARALLEL_GATEWAY` | AND split/join |
| `INCLUSIVE_GATEWAY` | OR (phase 2 OK) |
| `TIMER_CATCH` | Wait duration/date |
| `MESSAGE_CATCH` | Wait signal/message |
| `CALL_ACTIVITY` | Subprocess |
| `SEND_TASK` | Emit notification/event |

### 4.3 Instance lifecycle

```text
CREATED → RUNNING → (SUSPENDED) → COMPLETED | CANCELLED | FAULTED | COMPENSATING → COMPENSATED
```

### 4.4 Work item lifecycle

```text
CREATED → READY → RESERVED (claimed/assignee)
                 → COMPLETED | REJECTED | DELEGATED | REASSIGNED
                 → EXPIRED | CANCELLED
```

### 4.5 Agent resolution order (example)

1. Explicit assignee variable  
2. Node agent rule (ROLE `FINANCE_APPROVER`)  
3. Manager of initiator (org)  
4. Fallback queue  
5. Unresolved → `UNASSIGNED` queue + alert  

### 4.6 Business correlation

```text
business_type = "sales.order"
business_id   = "<uuid>"
business_key  = "BILTY:BL/…/000148:CANCEL"
```

Start with `unique_active=true` rejects second cancel approval while one RUNNING.

### 4.7 Variables

Typed map: `amount`, `reason_code`, `initiator_id`, `company_id`, …  
Schema on definition; unknown keys rejected if strict mode.

---

## 5. SLA & escalation

| Level | Example |
|---|---|
| L1 | Reminder at 4h |
| L2 | Escalate to manager at 8h |
| L3 | Auto-reject / auto-approve policy at 24h (rare; explicit) |

Timers persisted; `p17_scheduler` or process worker polls due rows.

---

## 6. Delegation & absence

Users register substitutes with date range + process scope (all / keys).  
New READY items copy/route to substitute; audit shows original + acting user.

---

## 7. Integration patterns

### 7.1 Domain starts process

```text
POST /process/instances/start
  process_key, business_*, variables, idempotency_key
```

### 7.2 Domain listens outcome

Outbox: `process.instance.completed` with `outcome=APPROVED|REJECTED` + variables → domain applies effect.

### 7.3 Document release

p09 transition `require_process=true` → start `document.release.approval` → on complete p09 continues.

### 7.4 Rules gateway

Exclusive gateway condition `expr_key` → p11 evaluate → choose edge.

---

## 8. Security

### Permissions

| Code | Use |
|---|---|
| `process.definition.read` | Read defs |
| `process.definition.manage` | Edit/publish defs |
| `process.instance.start` | Start instances |
| `process.instance.read` | Read instances |
| `process.instance.cancel` | Cancel |
| `process.task.read` | Inbox read |
| `process.task.act` | Claim/complete/reject |
| `process.task.reassign` | Reassign admin |
| `process.task.delegate` | Delegation prefs |
| `process.admin` | Suspend, break, migrate |
| `process.audit.read` | Audit |
| `process.*` | Wildcard |

### RLS

FORCE RLS on tenant instances, tasks, comments, variables.

---

## 9. Module layout

```text
platforms/p10_process/
  application/
    services/
      definition_publisher.py
      runtime_engine.py
      token_executor.py
      work_item_service.py
      agent_resolver.py
      sla_escalation.py
      delegation.py
      signal_correlator.py
      compensation.py
    commands/… queries/…
    permissions/catalog.py
    handlers/   # registered service tasks
  domain/…
  infrastructure/
    http/… persistence/… messaging/outbox/
    gateways/ rules.py org.py notify.py document.py
  tests/unit/engine/ gateway/ sla/
```

---

## 10. Domain events

| Event | When |
|---|---|
| `process.definition.published` / `activated` | Catalog |
| `process.instance.started` / `completed` / `cancelled` / `faulted` | Runtime |
| `process.instance.suspended` / `resumed` | Ops |
| `process.task.created` / `assigned` / `completed` / `rejected` | Work items |
| `process.task.escalated` / `expired` | SLA |
| `process.signal.received` | Message catch |
| `process.compensation.started` / `completed` | Cancel paths |

Stream: `jesloterp:process:outbox`.

---

## 11. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, catalog, permissions |
| P2 | Definitions + linear user tasks |
| P3 | Inbox claim/complete + agents |
| P4 | XOR gateway + variables |
| P5 | Parallel gateway + join |
| P6 | SLA/escalation + delegation |
| P7 | Service tasks + signals |
| P8 | Subprocess + compensation |
| P9 | Packs (bilty cancel, credit, doc release) |
| P10 | Registry → **Live** |

---

## 12. Definition of Done (enterprise)

- [x] Pinned definition version on instance  
- [x] Parallel split/join correctness under concurrency  
- [x] Idempotent start & task complete  
- [x] Agent unresolved → fallback queue  
- [x] Escalation creates audit + reassignment  
- [x] Unique business_key prevents duplicate actives  
- [x] Service task allow-list only  
- [x] Tenant RLS GUCs on HTTP (`require_process_access` + persist/fetch)  
- [x] HTTP instances + inbox list empty catalog as `[]` on `AsyncSession` (not memory fallback)  
- [x] Cancel emits compensation path when modeled  
- [x] No cross-schema FKs  

---

## 13. Anti-patterns

| Don’t | Do |
|---|---|
| Hardcode approver user ids in domain | Agent rules / roles |
| Mutate bilty inside process tables | Service task → domain gateway |
| Edit ACTIVE definition in place | New version + activate |
| Email-only approval without work item | Always create task |
| Infinite script freedom | Registered handlers + p11 |
| Skip audit on reassign | Audit every action |

---

## 14. Related documents

- Schema: [`PROCESS_SCHEMA.md`](PROCESS_SCHEMA.md)  
- API: [`PROCESS_API.md`](PROCESS_API.md)  
- Rules: [`../11_rules/RULES_GUIDE.md`](../11_rules/RULES_GUIDE.md)  
- Document: [`../09_document/DOCUMENT_GUIDE.md`](../09_document/DOCUMENT_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
