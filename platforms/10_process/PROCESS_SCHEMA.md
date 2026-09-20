# JeslotERP Process Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-09  
**Status:** **SoR-Live** — 66 ORM tables on schema `process` (Alembic `c7d8e9f0a1b2` / RLS `d8e9f0a1b2c3`); HTTP instances/inbox persist on AsyncSession  
**Package:** `platforms.p10_process`  
**PostgreSQL schema:** `process`  
**Companion:** [`PROCESS_GUIDE.md`](PROCESS_GUIDE.md) · [`PROCESS_API.md`](PROCESS_API.md)

> Runtime models: `platforms/p10_process/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `process` (never `p10`) |
| Tables | `proc_*` |
| Soft delete | Definitions superseded; instances terminal states |
| Cross-schema | UUID + business_key only |
| RLS | FORCE on tenant instances/tasks |
| Graph | Nodes/edges JSON + normalized tables |
| Handlers | Allow-listed `handler_key` strings |

---

## 2. Complete table inventory (**63 tables**)

### 2.1 Catalog & definitions (11)

| # | Table | Purpose |
|---|---|---|
| 1 | `proc_process` | Process catalog (key, name) |
| 2 | `proc_category` | Categories |
| 3 | `proc_process_category` | M2M |
| 4 | `proc_definition` | Versioned definition header |
| 5 | `proc_definition_graph` | Serialized graph blob/JSON |
| 6 | `proc_node` | Normalized nodes |
| 7 | `proc_edge` | Normalized edges |
| 8 | `proc_variable_schema` | Declared variables |
| 9 | `proc_form_binding` | Task → form/metadata key |
| 10 | `proc_service_handler` | Allow-listed service tasks |
| 11 | `proc_feature_binding` | Feature flags |

### 2.2 Agent & org routing (7)

| # | Table | Purpose |
|---|---|---|
| 12 | `proc_agent_rule` | Agent determination rules |
| 13 | `proc_agent_rule_step` | Ordered strategies |
| 14 | `proc_queue` | Work queues |
| 15 | `proc_queue_member` | Queue membership |
| 16 | `proc_role_binding` | Process role → IAM role |
| 17 | `proc_fallback_policy` | Unresolved agent policy |
| 18 | `proc_initiator_policy` | Who may start |

### 2.3 Runtime instances (9)

| # | Table | Purpose |
|---|---|---|
| 19 | `proc_instance` | Process instance |
| 20 | `proc_instance_variable` | Variables |
| 21 | `proc_token` | Execution tokens |
| 22 | `proc_token_history` | Token moves |
| 23 | `proc_correlation` | Business key correlation |
| 24 | `proc_instance_link` | Links to entities/docs |
| 25 | `proc_instance_suspension` | Suspend records |
| 26 | `proc_incident` | Faults / retries |
| 27 | `proc_compensation_log` | Compensation actions |

### 2.4 Work items / tasks (10)

| # | Table | Purpose |
|---|---|---|
| 28 | `proc_work_item` | Human tasks |
| 29 | `proc_work_item_assignee` | Assignees / candidates |
| 30 | `proc_work_item_action` | Complete payloads audit |
| 31 | `proc_work_item_comment` | Comments |
| 32 | `proc_work_item_attachment` | media_id refs |
| 33 | `proc_work_item_claim` | Claim history |
| 34 | `proc_decision_option` | Approve/Reject/… catalog per node |
| 35 | `proc_adhoc_task` | Side tasks |
| 36 | `proc_task_view_pref` | Inbox UI prefs |
| 37 | `proc_inbox_snapshot` | Optional denorm inbox |

### 2.5 SLA / timers / escalation (7)

| # | Table | Purpose |
|---|---|---|
| 38 | `proc_sla_policy` | SLA definitions |
| 39 | `proc_sla_binding` | Bind to node/process |
| 40 | `proc_deadline` | Concrete deadlines |
| 41 | `proc_escalation_level` | Ladder |
| 42 | `proc_escalation_event` | Fired escalations |
| 43 | `proc_timer_subscription` | Timer catch waiting |
| 44 | `proc_timer_fire` | Fire history |

### 2.6 Delegation, signals, subprocess (8)

| # | Table | Purpose |
|---|---|---|
| 45 | `proc_delegation` | OOO substitutes |
| 46 | `proc_delegation_scope` | Process key scope |
| 47 | `proc_message_def` | Message/signal catalog |
| 48 | `proc_message_subscription` | Waiting catches |
| 49 | `proc_message_correlation` | Inbound messages |
| 50 | `proc_subprocess_bind` | Call activity links |
| 51 | `proc_signal_audit` | Signal receipt audit |
| 52 | `proc_reassign_reason` | Reason codes |

### 2.7 Governance, packs, metrics (6)

| # | Table | Purpose |
|---|---|---|
| 53 | `proc_changeset` | Definition changes |
| 54 | `proc_approval` | Def publish approvals |
| 55 | `proc_package` | Process packs |
| 56 | `proc_package_item` | Pack contents |
| 57 | `proc_usage_stats` | Throughput metrics |
| 58 | `proc_migrate_plan` | Instance migration plans |

### 2.8 Audit & plumbing (5 +)

| # | Table | Purpose |
|---|---|---|
| 59 | `proc_action_audit` | Generic action audit |
| 60 | `proc_definition_diff` | Version diffs |
| 61 | `proc_simulation_run` | Dry-run simulations |
| 62 | `proc_notification_hook` | Notify templates binding |
| 63 | `proc_outcome_code` | Standard outcomes |

**Plumbing:** `proc_outbox`, `proc_idempotency_key`, `proc_catalog_audit`

**Implementation total with plumbing: 66 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `proc_def_lifecycle` | `DRAFT`, `PUBLISHED`, `ACTIVE`, `RETIRED` |
| `proc_instance_status` | `CREATED`, `RUNNING`, `SUSPENDED`, `COMPLETED`, `CANCELLED`, `FAULTED`, `COMPENSATING`, `COMPENSATED` |
| `proc_token_status` | `ACTIVE`, `WAITING`, `CONSUMED`, `CANCELLED` |
| `proc_node_kind` | `START`, `END`, `END_TERMINATE`, `USER_TASK`, `SERVICE_TASK`, `SCRIPT_TASK`, `EXCLUSIVE_GATEWAY`, `PARALLEL_GATEWAY`, `INCLUSIVE_GATEWAY`, `TIMER_CATCH`, `MESSAGE_CATCH`, `CALL_ACTIVITY`, `SEND_TASK` |
| `proc_work_status` | `CREATED`, `READY`, `RESERVED`, `COMPLETED`, `REJECTED`, `CANCELLED`, `EXPIRED`, `DELEGATED` |
| `proc_agent_strategy` | `USER`, `ROLE`, `QUEUE`, `INITIATOR`, `INITIATOR_MANAGER`, `ORG_PATH`, `VARIABLE`, `HANDLER`, `FALLBACK` |
| `proc_escalation_action` | `NOTIFY`, `REASSIGN_MANAGER`, `REASSIGN_ROLE`, `AUTO_COMPLETE`, `AUTO_REJECT`, `CANCEL_INSTANCE` |
| `proc_incident_status` | `OPEN`, `RETRYING`, `RESOLVED`, `IGNORED` |
| `proc_outcome` | `APPROVED`, `REJECTED`, `CANCELLED`, `TIMED_OUT`, `CUSTOM` |

---

## 4. Definitions (detail)

### 4.1 `proc_process`

| Column | Type | Notes |
|---|---|---|
| `process_key` | VARCHAR(100) UNIQUE | `sales.order.cancel.approval` |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `label_key` | VARCHAR(200) NULL | i18n |
| `owner_team` | VARCHAR(100) NULL | |
| `is_active` | BOOLEAN | |
| `unique_active_per_business_key` | BOOLEAN | Default correlation policy |

### 4.2 `proc_definition`

| Column | Type | Notes |
|---|---|---|
| `process_id` | UUID | |
| `version_number` | INT | |
| `lifecycle` | VARCHAR(20) | |
| `checksum` | VARCHAR(64) | |
| `published_at` | TIMESTAMPTZ NULL | |
| `activated_at` | TIMESTAMPTZ NULL | |
| `strict_variables` | BOOLEAN | |
| `notes` | TEXT NULL | |

**Unique:** `(process_id, version_number)`. At most one ACTIVE per process.

### 4.3 `proc_node`

| Column | Type | Notes |
|---|---|---|
| `definition_id` | UUID | |
| `node_key` | VARCHAR(80) | Stable in graph |
| `kind` | VARCHAR(30) | |
| `name` | VARCHAR(150) | |
| `assignee_rule_id` | UUID NULL | |
| `form_key` | VARCHAR(100) NULL | metadata form |
| `handler_key` | VARCHAR(100) NULL | service task |
| `sla_policy_id` | UUID NULL | |
| `multi_instance` | VARCHAR(20) NULL | NONE/PARALLEL/SEQUENTIAL |
| `join_type` | VARCHAR(20) NULL | for gateways |
| `config` | JSONB | node-specific |

### 4.4 `proc_edge`

| Column | Type | Notes |
|---|---|---|
| `definition_id` | UUID | |
| `from_node_id` | UUID | |
| `to_node_id` | UUID | |
| `edge_key` | VARCHAR(80) | |
| `condition_type` | VARCHAR(30) | ALWAYS, OUTCOME, EXPR, RULE_KEY, DEFAULT |
| `condition_value` | VARCHAR(200) NULL | `APPROVED` / rule key |
| `priority` | INT | XOR evaluation order |

---

## 5. Agent rules

### 5.1 `proc_agent_rule_step`

| Column | Type | Notes |
|---|---|---|
| `rule_id` | UUID | |
| `position` | INT | |
| `strategy` | VARCHAR(30) | |
| `strategy_value` | VARCHAR(200) NULL | role code / queue key / var name |
| `stop_if_resolved` | BOOLEAN DEFAULT true | |

---

## 6. Instances & tokens

### 6.1 `proc_instance`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | RLS |
| `company_id` | UUID NULL | |
| `process_id` | UUID | |
| `definition_id` | UUID | Pinned version |
| `status` | VARCHAR(20) | |
| `business_type` | VARCHAR(100) NULL | |
| `business_id` | UUID NULL | |
| `business_key` | VARCHAR(200) NULL | |
| `outcome` | VARCHAR(30) NULL | |
| `initiated_by` | UUID NULL | |
| `started_at` / `ended_at` | TIMESTAMPTZ | |
| `parent_instance_id` | UUID NULL | Subprocess |
| `idempotency_key` | VARCHAR(100) NULL | |

### 6.2 `proc_correlation`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `process_id` | UUID | |
| `business_key` | VARCHAR(200) | |
| `instance_id` | UUID | |
| `is_active` | BOOLEAN | |

**Partial unique:** active `(tenant_id, process_id, business_key)` when unique policy.

### 6.3 `proc_token`

| Column | Type | Notes |
|---|---|---|
| `instance_id` | UUID | |
| `node_id` | UUID | Current node |
| `status` | VARCHAR(20) | |
| `parent_token_id` | UUID NULL | Parallel tree |
| `waiting_on` | VARCHAR(30) NULL | TASK/TIMER/MESSAGE/JOIN |

### 6.4 `proc_instance_variable`

| Column | Type | Notes |
|---|---|---|
| `instance_id` | UUID | |
| `var_name` | VARCHAR(80) | |
| `value_json` | JSONB | |
| `value_type` | VARCHAR(20) | |
| `is_sensitive` | BOOLEAN | Redact in APIs |

---

## 7. Work items

### 7.1 `proc_work_item`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `instance_id` | UUID | |
| `token_id` | UUID | |
| `node_id` | UUID | |
| `task_key` | VARCHAR(80) | |
| `title` | VARCHAR(200) | |
| `status` | VARCHAR(20) | |
| `priority` | INT | |
| `form_key` | VARCHAR(100) NULL | |
| `queue_id` | UUID NULL | |
| `reserved_by` | UUID NULL | |
| `reserved_at` | TIMESTAMPTZ NULL | |
| `due_at` | TIMESTAMPTZ NULL | |
| `outcome` | VARCHAR(30) NULL | |
| `completed_at` | TIMESTAMPTZ NULL | |
| `acting_user_id` | UUID NULL | Completer (may be substitute) |
| `original_assignee_id` | UUID NULL | |

### 7.2 `proc_work_item_assignee`

| Column | Type | Notes |
|---|---|---|
| `work_item_id` | UUID | |
| `principal_type` | VARCHAR(20) | USER/ROLE/QUEUE |
| `principal_id` | UUID NULL | |
| `queue_id` | UUID NULL | |
| `is_candidate` | BOOLEAN | |
| `is_owner` | BOOLEAN | |

---

## 8. SLA / timers

### 8.1 `proc_deadline`

| Column | Type | Notes |
|---|---|---|
| `work_item_id` | UUID NULL | |
| `timer_subscription_id` | UUID NULL | |
| `due_at` | TIMESTAMPTZ | |
| `status` | VARCHAR(20) | OPEN/FIRED/CANCELLED |
| `escalation_level` | INT | |

### 8.2 `proc_escalation_level`

| Column | Type | Notes |
|---|---|---|
| `sla_policy_id` | UUID | |
| `level_no` | INT | |
| `offset_seconds` | INT | From clock start |
| `action` | VARCHAR(30) | |
| `action_value` | VARCHAR(200) NULL | role/queue |

---

## 9. Delegation & messages

### 9.1 `proc_delegation`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `user_id` | UUID | Absent user |
| `substitute_user_id` | UUID | |
| `valid_from` / `valid_to` | TIMESTAMPTZ | |
| `is_active` | BOOLEAN | |

### 9.2 `proc_message_subscription`

| Column | Type | Notes |
|---|---|---|
| `instance_id` | UUID | |
| `token_id` | UUID | |
| `message_key` | VARCHAR(100) | |
| `correlation_value` | VARCHAR(200) | |
| `status` | VARCHAR(20) | WAITING/CONSUMED |

---

## 10. Incidents & compensation

### 10.1 `proc_incident`

| Column | Type | Notes |
|---|---|---|
| `instance_id` | UUID | |
| `token_id` | UUID NULL | |
| `error_code` | VARCHAR(50) | |
| `error_detail` | JSONB | |
| `status` | VARCHAR(20) | |
| `retry_count` | INT | |

### 10.2 `proc_compensation_log`

Records compensation handler executions for cancelled instances.

---

## 11. Governance & packs

- Changeset/approval before activate definition  
- Packages: `sales.order.cancel@1.0.0`, `finance.credit.override@1.0.0`, `document.release@1.0.0`  
- Simulation runs for admin dry-walk of XOR paths  

---

## 12. Plumbing

| Table | Purpose |
|---|---|
| `proc_outbox` | Domain events |
| `proc_idempotency_key` | Start/complete/signal |
| `proc_catalog_audit` | Definition audit |

---

## 13. RLS summary

| Class | Policy |
|---|---|
| Process catalog / handlers | Read auth; manage permission |
| Instances, tokens, variables, tasks | FORCE `tenant_id` |
| Queues | Tenant-scoped membership |
| Delegations | Tenant + self/manage |

---

## 14. Seed minimum

1. Outcomes: APPROVED, REJECTED, CANCELLED  
2. Handlers: `domain.noop`, `document.continue_transition`, `notify.task_assigned`  
3. Queues: `UNASSIGNED`, `FINANCE`, `OPERATIONS`  
4. Processes: `sales.order.cancel.approval`, `credit.limit.override`, `document.release.approval`  
5. Linear definitions ACTIVE for each  
6. SLA default 4h/8h notify/escalate  
7. Permissions `process.*`  

---

## 15. ER overview

```text
proc_process ── definitions ── nodes / edges / variable_schema
                     │
                     ▼
              instances ── tokens ── work_items ── assignees
                     │         │
                     │         ├── deadlines / escalations
                     │         └── message/timer subscriptions
                     ├── variables / correlation / links
                     └── incidents / compensation

agent_rules / queues / delegations
packages / changesets
```

---

## 16. Implementation notes

1. Engine advances in short DB transactions; long service tasks async with incidents.  
2. Parallel join waits until sibling tokens arrive or terminate.  
3. Never delete COMPLETED work items — status only.  
4. Sensitive variables redacted in list APIs.  
5. Split models: `catalog`, `graph`, `runtime`, `task`, `sla`, `delegation`, `signal`, `governance`, `plumbing`.
