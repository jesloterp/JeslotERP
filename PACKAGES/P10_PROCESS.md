# Process (`p10_process`)

**Package:** `p10_process`  
**Schema:** `process`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Workflow and approval control plane: definitions, instances, inbox tasks, SLA, and delegation.

## 2. Responsibilities

- Own the `process` persistence schema and the `p10_process` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Definition administration and publish
- Instance start, signal, cancel, and suspend
- Inbox claim, complete, reject, reassign, and delegate
- Internal durable start
- SLA and delegation models
- Simulation hooks in design

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Process definition**
- **Versioned graph**
- **Instance**
- **Work item / task**
- **Queue / agent rule**
- **SLA / deadline**
- **Delegation**
- **Business key**

## 5. Major Capabilities

- Definition administration and publish
- Instance start, signal, cancel, and suspend
- Inbox claim, complete, reject, reassign, and delegate
- Internal durable start
- SLA and delegation models
- Simulation hooks in design

## 6. Public Architecture

```text
HTTP / internal API
        ↓
Application services (commands, queries, ports)
        ↓
Domain concepts and policies
        ↓
Adapters (persistence, optional providers, outbox)
```

Conceptual tables (purpose only):

- Catalog / graph
- Instances / tokens
- Tasks / inbox
- Agents / queues
- SLA / delegation
- Governance
- Outbox

Public API resource groups:

- Admin definitions
- Instances
- Tasks / inbox
- Internal engine

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p03_configuration`
- `p04_business_partner`
- `p09_document`

### Additional verified runtime coupling

- Record sharing via p33_sharing
- Extensibility hooks on lifecycle

### Recommended future dependency

- p11_rules for gateway conditions
- p17_scheduler for timers
- p15_notification for task alerts
- p14_messaging for service tasks

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Future finance close, sales credit, purchase release, and any approval-bound document

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `process.definition.published`
- `process.instance.started`
- `process.instance.suspended`
- `process.task.created`
- `process.task.completed`
- `process.task.rejected`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Unique-active-per-business-key and handler allow-lists.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- process permissions
- RLS
- Fail-closed task ACL
- Sensitive variable redaction

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Service-handler registry
- Feature bindings
- Packs

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `process` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Dedicated queue tables and SLA metrics are pending
- Full BPMN engine breadth is not verified
- Timer integration with the scheduler is recommended, not fully verified end-to-end
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Approvals, exceptions, and multi-step releases are platform concerns, not boolean columns on business tables.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
