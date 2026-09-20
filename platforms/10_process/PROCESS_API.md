# JeslotERP Process Platform — Complete API Specification (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-09  
**Status:** **SoR-Live** — public `/api/v1/process` + internal `/internal/v1/process`; instances + inbox Postgres-first when session is AsyncSession  
**Package:** `platforms.p10_process`  
**PostgreSQL schema:** `process`  
**Public base:** `/api/v1/process`  
**Internal base:** `/internal/v1/process`  
**AuthN:** Bearer JWT · **Internal:** `X-Internal-Token`  
**Companion:** [`PROCESS_GUIDE.md`](PROCESS_GUIDE.md) · [`PROCESS_SCHEMA.md`](PROCESS_SCHEMA.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Start/signal/cancel instances, inbox claim/complete, definitions publish, agents/queues, SLA, delegation, subprocess, compensation, packs, internal engine ticks. |

---

## 1. Design principles (advanced)

1. **Start & act are primary** — domain uses `/instances/start` and `/tasks/*`; graph authoring is admin.  
2. **Pinned definition version** — runtime never silently switches graph mid-flight.  
3. **Idempotent start/complete/signal** — `Idempotency-Key` required.  
4. **Inbox-centric UX** — claim/complete/reject/reassign/delegate.  
5. **Fail closed on ACL** — only candidates/assignees/admins act on tasks.  
6. **Unique correlation** — honor process `unique_active_per_business_key`.  
7. **Allow-listed service handlers** — unknown `handler_key` → incident.  
8. **SLA visible** — due_at + escalation events on task.  
9. **Outcomes standardized** — APPROVED/REJECTED/… plus custom when declared.  
10. **Internal tick APIs** — timers/escalations for workers/scheduler.  
11. **Sensitive variables redacted** in list responses.  
12. **Simulation** — dry-run XOR paths without side effects.

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
    "idempotent_replay": false
  }
}
```

---

## 4. Errors

```text
PROC_PROCESS_NOT_FOUND / DEFINITION_NOT_FOUND / DEFINITION_NOT_ACTIVE
PROC_INSTANCE_NOT_FOUND / INSTANCE_NOT_RUNNING / ALREADY_COMPLETED
PROC_DUPLICATE_ACTIVE / CORRELATION_CONFLICT
PROC_TASK_NOT_FOUND / TASK_NOT_READY / TASK_NOT_RESERVED / TASK_FORBIDDEN
PROC_CLAIM_CONFLICT / INVALID_OUTCOME
PROC_TRANSITION_INVALID / GATEWAY_UNRESOLVED
PROC_HANDLER_UNKNOWN / HANDLER_FAILED
PROC_SIGNAL_UNMATCHED / TIMER_NOT_DUE
PROC_DELEGATION_INVALID
PROC_VARIABLE_INVALID / VARIABLE_REQUIRED
PROC_PUBLISH_CONFLICT / APPROVAL_REQUIRED
PROC_MIGRATE_FORBIDDEN
PROC_IDEMPOTENCY_CONFLICT / VERSION_CONFLICT
PROC_PACKAGE_CHECKSUM_MISMATCH
PROC_INCIDENT_OPEN
```

HTTP: `404` · `409` · `422` · `403` · `412`.

---

## 5. Permissions

| Code | Use |
|---|---|
| `process.definition.read` | Read catalog/defs |
| `process.definition.manage` | Edit/publish |
| `process.instance.start` | Start |
| `process.instance.read` | Read instances |
| `process.instance.cancel` | Cancel |
| `process.task.read` | Inbox |
| `process.task.act` | Claim/complete/reject |
| `process.task.reassign` | Reassign |
| `process.task.delegate` | Delegation prefs |
| `process.admin` | Suspend, incidents, migrate |
| `process.audit.read` | Audit |
| `process.*` | All |

---

## 6. Runtime — instances (primary for domain)

### 6.1 Start

```http
POST /api/v1/process/instances/start
Idempotency-Key: …
```

```json
{
  "process_key": "sales.order.cancel.approval",
  "company_id": "…",
  "business_type": "sales.order",
  "business_id": "…",
  "business_key": "BILTY:…:CANCEL",
  "variables": {
    "amount": 125000.50,
    "reason_code": "CUSTOMER_REQUEST",
    "bilty_number": "BL/MH01/2526/000148"
  },
  "links": [
    { "entity_type": "sales.order", "entity_id": "…" },
    { "entity_type": "document.document", "entity_id": "…" }
  ]
}
```

**Response:**

```json
{
  "instance_id": "…",
  "definition_id": "…",
  "status": "RUNNING",
  "business_key": "BILTY:…:CANCEL",
  "open_tasks": [
    {
      "work_item_id": "…",
      "task_key": "manager_approve",
      "status": "READY",
      "due_at": "…"
    }
  ]
}
```

Duplicate active → `409 PROC_DUPLICATE_ACTIVE`.

### 6.2 Get / list instances

```http
GET /api/v1/process/instances/{instance_id}
GET /api/v1/process/instances?process_key=…&status=RUNNING&business_id=…
GET /api/v1/process/instances/by-business?business_type=sales.order&business_id=…
```

### 6.3 Variables

```http
GET /api/v1/process/instances/{instance_id}/variables
PATCH /api/v1/process/instances/{instance_id}/variables   # admin / restricted keys only
```

### 6.4 Cancel / suspend / resume

```http
POST /api/v1/process/instances/{instance_id}/cancel
POST /api/v1/process/instances/{instance_id}/suspend
POST /api/v1/process/instances/{instance_id}/resume
```

**Cancel body:** `{ "reason": "…", "compensate": true }`

### 6.5 Signal / message

```http
POST /api/v1/process/signals
Idempotency-Key: …
```

```json
{
  "message_key": "payment.received",
  "correlation_value": "INV-…",
  "payload": { "amount": 1000 }
}
```

Matches waiting `MESSAGE_CATCH` subscriptions.

---

## 7. Inbox & work items (primary for users)

### 7.1 My inbox

```http
GET /api/v1/process/inbox
GET /api/v1/process/inbox?queue=FINANCE&status=READY
GET /api/v1/process/inbox/count
```

Returns candidate + assigned tasks for caller (including delegation).

### 7.2 Get task

```http
GET /api/v1/process/tasks/{work_item_id}
```

Includes form_key, variables (redacted), decision options, due_at, instance summary.

### 7.3 Claim / unclaim

```http
POST /api/v1/process/tasks/{work_item_id}/claim
POST /api/v1/process/tasks/{work_item_id}/unclaim
```

Claim → RESERVED by caller; conflict if already reserved.

### 7.4 Complete / reject

```http
POST /api/v1/process/tasks/{work_item_id}/complete
Idempotency-Key: …
POST /api/v1/process/tasks/{work_item_id}/reject
Idempotency-Key: …
```

**Complete body:**

```json
{
  "outcome": "APPROVED",
  "comment": "Within policy",
  "variables": {
    "finance_note": "OK"
  },
  "form_payload": { "checkbox_ack": true }
}
```

Engine validates outcome against node options; advances tokens; may create next tasks.

### 7.5 Reassign / delegate action

```http
POST /api/v1/process/tasks/{work_item_id}/reassign
POST /api/v1/process/tasks/{work_item_id}/delegate
```

**Reassign:** `{ "to_user_id": "…", "reason_code": "WORKLOAD" }` — requires `process.task.reassign`.

### 7.6 Comments & attachments

```http
GET  /api/v1/process/tasks/{work_item_id}/comments
POST /api/v1/process/tasks/{work_item_id}/comments
POST /api/v1/process/tasks/{work_item_id}/attachments
```

Attachments: `{ "media_id": "…" }` (p08).

### 7.7 Ad-hoc tasks

```http
POST /api/v1/process/instances/{instance_id}/adhoc-tasks
POST /api/v1/process/adhoc-tasks/{id}/complete
```

---

## 8. Delegation (OOO)

```http
GET  /api/v1/process/delegations/me
PUT  /api/v1/process/delegations/me
GET  /api/v1/process/delegations          # admin
```

```json
{
  "substitute_user_id": "…",
  "valid_from": "2026-09-10T00:00:00Z",
  "valid_to": "2026-09-20T00:00:00Z",
  "process_keys": ["sales.order.cancel.approval"]
}
```

Empty `process_keys` = all processes.

---

## 9. Queues

```http
GET  /api/v1/process/queues
POST /api/v1/process/queues
GET  /api/v1/process/queues/{queue_key}/members
PUT  /api/v1/process/queues/{queue_key}/members
GET  /api/v1/process/queues/{queue_key}/inbox
```

---

## 10. Definitions (admin)

### 10.1 Catalog

```http
GET  /api/v1/process/processes
POST /api/v1/process/processes
GET  /api/v1/process/processes/{process_key}
PATCH /api/v1/process/processes/{process_key}
```

### 10.2 Versions & graph

```http
GET  /api/v1/process/processes/{process_key}/definitions
POST /api/v1/process/processes/{process_key}/definitions
GET  /api/v1/process/definitions/{definition_id}
PUT  /api/v1/process/definitions/{definition_id}/graph
GET  /api/v1/process/definitions/{definition_id}/graph
PUT  /api/v1/process/definitions/{definition_id}/variables
PUT  /api/v1/process/definitions/{definition_id}/agent-rules
```

**Graph put (normalized):**

```json
{
  "nodes": [
    { "node_key": "start", "kind": "START" },
    { "node_key": "mgr", "kind": "USER_TASK", "form_key": "approval.simple", "assignee_rule_key": "initiator_manager" },
    { "node_key": "xor_amt", "kind": "EXCLUSIVE_GATEWAY" },
    { "node_key": "fin", "kind": "USER_TASK", "assignee_rule_key": "role.FINANCE_APPROVER" },
    { "node_key": "apply", "kind": "SERVICE_TASK", "handler_key": "sales.order.apply_cancel" },
    { "node_key": "end", "kind": "END" }
  ],
  "edges": [
    { "from": "start", "to": "mgr", "condition_type": "ALWAYS" },
    { "from": "mgr", "to": "xor_amt", "condition_type": "OUTCOME", "condition_value": "APPROVED" },
    { "from": "mgr", "to": "end", "condition_type": "OUTCOME", "condition_value": "REJECTED" },
    { "from": "xor_amt", "to": "fin", "condition_type": "RULE_KEY", "condition_value": "amount_gt_100k", "priority": 1 },
    { "from": "xor_amt", "to": "apply", "condition_type": "DEFAULT", "priority": 100 },
    { "from": "fin", "to": "apply", "condition_type": "OUTCOME", "condition_value": "APPROVED" },
    { "from": "apply", "to": "end", "condition_type": "ALWAYS" }
  ]
}
```

Only DRAFT definitions mutable.

### 10.3 Publish / activate

```http
POST /api/v1/process/definitions/{definition_id}/publish
POST /api/v1/process/definitions/{definition_id}/activate
POST /api/v1/process/definitions/{definition_id}/retire
```

Activate retires previous ACTIVE (policy).

### 10.4 Simulate

```http
POST /api/v1/process/definitions/{definition_id}/simulate
```

```json
{
  "variables": { "amount": 150000 },
  "outcomes": { "mgr": "APPROVED", "fin": "APPROVED" }
}
```

Returns visited node path; no side effects.

---

## 11. SLA, timers, incidents

```http
GET  /api/v1/process/sla-policies
POST /api/v1/process/sla-policies
GET  /api/v1/process/tasks/{work_item_id}/escalations
GET  /api/v1/process/instances/{instance_id}/incidents
POST /api/v1/process/incidents/{incident_id}/retry
POST /api/v1/process/incidents/{incident_id}/resolve
```

---

## 12. Service handlers registry

```http
GET  /api/v1/process/handlers
POST /api/v1/process/handlers          # process.admin — register metadata only
```

Handlers are implemented in code; API registers keys + description + input schema.

---

## 13. Packages & governance

```http
GET  /api/v1/process/packages
POST /api/v1/process/packages/{package_key}/install
GET  /api/v1/process/changesets
POST /api/v1/process/changesets
POST /api/v1/process/changesets/{id}/approvals
```

---

## 14. Migration (advanced ops)

```http
POST /api/v1/process/migrations
GET  /api/v1/process/migrations/{id}
POST /api/v1/process/migrations/{id}/execute
```

Migrate running instances from def vN → vN+1 with explicit mapping; forbidden by default without plan.

---

## 15. Audit & metrics

```http
GET /api/v1/process/instances/{instance_id}/audit
GET /api/v1/process/tasks/{work_item_id}/audit
GET /api/v1/process/stats?process_key=…&from=…&to=…
```

Requires `process.audit.read` for audit.

---

## 16. Internal / worker APIs

| Endpoint | Purpose |
|---|---|
| `POST /internal/v1/process/timers/tick` | Fire due timers/deadlines |
| `POST /internal/v1/process/escalations/tick` | Apply escalation ladder |
| `POST /internal/v1/process/service-tasks/callback` | Async handler result |
| `POST /internal/v1/process/instances/start` | Trusted domain start |
| `POST /internal/v1/process/signals` | Trusted signal |
| `GET  /internal/v1/process/instances/{id}/outcome` | Domain poll outcome |
| `GET  /internal/v1/process/health` | Engine health |

Internal auth: `X-Internal-Token`.

---

## 17. Caching & concurrency

| Resource | Strategy |
|---|---|
| ACTIVE definition | Cached per process_key; invalidate on activate |
| Task claim | Conditional update status READY→RESERVED |
| Token advance | Row lock on instance/token |
| Start uniqueness | Partial unique correlation |
| Idempotency | Durable keys for start/complete/signal |

---

## 18. Example client flows

### 18.1 Bilty cancel approval

1. Domain validates cancel eligibility  
2. `POST /instances/start` process `sales.order.cancel.approval` + business_key  
3. Manager completes APPROVED  
4. XOR → finance if amount high  
5. Service task `sales.order.apply_cancel`  
6. Domain receives `process.instance.completed` outcome APPROVED  

### 18.2 Document release

1. p09 transition requires process  
2. Start `document.release.approval`  
3. On COMPLETED APPROVED → p09 continues RELEASED  

### 18.3 Inbox day

1. `GET /inbox`  
2. `claim` → review form → `complete`  
3. Delegation covers absence automatically  

### 18.4 Stuck service task

1. Incident OPEN  
2. Admin `retry` or `resolve`  
3. Engine continues or cancels  

---

## 19. Event hooks

| Event | Consumer |
|---|---|
| `process.instance.completed` | Domain apply / unlock |
| `process.task.created` / `assigned` | Notification |
| `process.task.escalated` | Manager alert |
| `process.instance.faulted` | Ops |
| `process.definition.activated` | Cache bust |

---

## 20. Compatibility notes

- Public prefix `/api/v1/process`; schema `process`.  
- Conditions may call p11 via `RULE_KEY`; simple OUTCOME/DEFAULT need no rules.  
- UI forms referenced by `form_key` from metadata — payloads validated lightly in p10.  
- Never expose other tenants’ inbox via queue misconfiguration — membership RLS.

---

## 21. Related documents

- Guide: [`PROCESS_GUIDE.md`](PROCESS_GUIDE.md)  
- Schema: [`PROCESS_SCHEMA.md`](PROCESS_SCHEMA.md)  
- Rules: [`../11_rules/RULES_API.md`](../11_rules/RULES_API.md)  
- Document: [`../09_document/DOCUMENT_API.md`](../09_document/DOCUMENT_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
