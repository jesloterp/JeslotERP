# Workflow / Process Engine

**Package:** `p10_process`  
**Status:** PARTIALLY_IMPLEMENTED  
**Kernel:** SoR-Live (instances and inbox persist)  
**Production:** withheld

## Purpose

`p10_process` is the workflow and approval plane. It is not a boolean
`approved` column and it is not the rules engine.

## Current capability

- Versioned process definitions and publish / activate.
- Instance start, signal, cancel, suspend.
- Inbox: claim, complete, reject, reassign, delegate.
- Business-key correlation (unique active instance when configured).
- SLA / delegation models exist in the schema and services.
- Record sharing on instances.
- Extensibility hooks on lifecycle.
- Outbox events for definition and task lifecycle.

## Planned / incomplete

- Dedicated queue tables and SLA metrics.
- Full BPMN breadth (every gateway, compensation, subprocess) is **not** verified as a complete engine.
- Timer due-work fully owned by `p17_scheduler` is recommended, not fully verified.
- Rules evaluation on exclusive gateways is the intended split, not a proven universal path.
- Production label.

## Concepts

| Concept | Meaning |
|---|---|
| Process definition | Versioned graph |
| Instance | Running or completed case |
| Token | Where the flow is |
| Work item | Human inbox task |
| Agent rule | Who should act |
| SLA | Deadline / escalation policy |
| Delegation | Substitute coverage |
| Business key | Correlation to a domain document |

## Intended cycle

```text
Domain API / event
    → start instance
    → user task → inbox → notification (recommended)
    → exclusive gateway → p11_rules (recommended)
    → service task → domain gateway
    → timer → scheduler / worker (recommended)
    → end → outbox callback
```

Domain modules start processes and react to outcomes. The engine does not own
invoice or stock tables.

## Auditability

Task history and outbox events are the public audit story. Deep forensic
payloads belong in `p19_audit` when emitters integrate.

## Roles

Agent resolution is intended to support user, role, group, manager, org path,
and partner contact strategies. Treat unproven strategies as `NOT_VERIFIED`.
