# Scheduler (`p17_scheduler`)

**Package:** `p17_scheduler`  
**Schema:** `scheduler`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Enterprise scheduling: cron, calendars, blackouts, misfire / overlap policy, run ledger, and distributed tick.

## 2. Responsibilities

- Own the `scheduler` persistence schema and the `p17_scheduler` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Schedules, versions, runs, dependencies, simulate, run-now
- Calendars, blackouts, policies
- Ops quotas, tickers, packs
- Internal tick and run callbacks
- Persistence-backed calendars and ticker / lock records
- Optional distributed tick lock

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Schedule / version**
- **Calendar day**
- **Blackout**
- **Misfire / overlap policy**
- **Run**
- **Tick lock**
- **Schedule dependency**

## 5. Major Capabilities

- Schedules, versions, runs, dependencies, simulate, run-now
- Calendars, blackouts, policies
- Ops quotas, tickers, packs
- Internal tick and run callbacks
- Persistence-backed calendars and ticker / lock records
- Optional distributed tick lock

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

- Schedules / versions / policies
- Calendars / blackouts
- Runs / next-fire
- Ticker / locks
- Bridge
- Governance

Public API resource groups:

- Schedules
- Calendars / policies
- Ops
- Internal tick

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p14_messaging`

### Additional verified runtime coupling

- Enqueue bridge to p14_messaging

### Recommended future dependency

- p02_organization calendars
- p15_notification
- p21_monitoring

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Licensing dunning / compile schedules
- future period-close and MRP runs

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `scheduler.schedule.activated`
- `scheduler.run.enqueued`
- `scheduler.misfire.detected`
- `scheduler.blackout.active`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Tick-lock attach is optional at startup.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- scheduler permissions
- Handler allow-list
- RLS

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Tick-lock port
- Messaging bridge
- Internal tick API

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `scheduler` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Live ticker soak evidence is environment-blocked
- Production multi-node fleet is not claimed

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Period close, depreciation, and recurring invoices are schedules, not cron hidden in app servers.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
