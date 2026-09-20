# JeslotERP Scheduler Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — HTTP calendars/blackouts + ticker/lock persist on `AsyncSession`. Tick hydrates calendar ledger from Postgres. Not Production.  
**Package:** `platforms.p17_scheduler`  
**PostgreSQL schema:** `scheduler`  
**Depends on:** `p01_identity`, `p14_messaging`  
**Integrates with:** `p02_organization` (timezones/fiscal calendars), `p03_configuration`, `p12_feature`, `p13_event_bus`, `p15_notification` (digest flush), `p16_cache` (warmup), `p21_monitoring`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`SCHEDULER_SCHEMA.md`](SCHEDULER_SCHEMA.md) · [`SCHEDULER_API.md`](SCHEDULER_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise scheduling plane: cron/calendars, timezones, misfire/overlap policies, blackout windows, dependencies, run ledger, distributed tick locks, catch-up, tenant schedules, messaging bridge. |
| 1.1 | 2026-09-12 | TASK-SOR-006: durable calendar + ticker/lock ledger; empty list is `[]`; RLS on calendar/ticker HTTP. |

---

## 1. Purpose (enterprise)

`p17_scheduler` is JeslotERP’s **time-based trigger control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Background Job Scheduling (SM36/SM37)** — definitions, start conditions, run history  
- **Microsoft Dynamics 365** — batch jobs / recurring processes with monitoring  
- **Salesforce Scheduled Apex / Cron triggers** — cron expressions, next-fire, abort  
- **Quartz / enterprise schedulers** — calendars, misfires, clustered locks  

It is **not** `while True: sleep(60)` in a random worker. It is the system that makes ERP timing correct for:

1. **Cron & calendar schedules** with IANA timezones  
2. **One-shot / delayed / windowed** fires  
3. **Misfire policies** — fire-now, skip, catch-up-limited  
4. **Overlap policies** — skip if running, queue next, forbid parallel  
5. **Blackout calendars** — month-end freeze, holidays  
6. **Schedule dependencies** — B after A succeeds  
7. **Durable run ledger** — who fired, outcome, correlation  
8. **Enqueue-only execution** — always hand off to **p14** handlers (no heavy inline work)  
9. **Cluster-safe ticking** — distributed locks, multi-instance  
10. **Tenant-scoped schedules** with quotas & manual run  

### Owns

| Domain | Examples |
|---|---|
| Schedule definitions | cron, calendars, one-shots |
| Calendars / blackouts | holiday, maintenance |
| Trigger policies | misfire, overlap |
| Run ledger | scheduled_run history |
| Tick engine | due computation, locks |
| Bridges | → p14 enqueue templates |
| Manual ops | trigger now, pause, resume |
| Catch-up | bounded backfill fires |
| Governance | packs, approvals |

### Does **not** own

| Concern | Owner |
|---|---|
| Job execution / retries / DLQ | `p14_messaging` |
| Domain event contracts | `p13_event_bus` |
| Fiscal year master data | `p02_organization` (referenced) |
| Notification send | `p15_notification` (schedule may flush digests) |
| BPM timers inside process instances | `p10_process` (may use scheduler for SLA ticks) |

### Critical split: Scheduler vs Messaging vs Process timers

| | **Scheduler (p17)** | **Messaging (p14)** | **Process (p10)** |
|---|---|---|---|
| Question | When should something start? | Do this work unit | Advance this workflow |
| Output | Enqueue job / emit tick event | Handler execution | Tokens / tasks |
| Clock | Cron/calendar | `run_at` delay on jobs | Deadline timers |

**Rule:** Scheduler ticks are cheap. Heavy work always becomes a p14 job.

---

## 2. Architectural position

```text
Cluster ticker(s)
    │  distributed lock
    ▼
due schedules ──► create scheduled_run
    │
    ├─ enqueue p14 job (handler_key + payload)
    └─ optional emit p13 event schedule.fired

p14 worker executes → updates run status via callback
```

**Hard rules**

1. **No business logic** inside ticker beyond enqueue/emit.  
2. Schedules pin `timezone` explicitly — never server-local implicit.  
3. Overlap policy enforced using run ledger + locks.  
4. No cross-schema FKs — UUID refs to org calendars.  
5. RLS fail-closed on tenant schedules/runs.  
6. Manual “run now” audited.

---

## 3. Advanced design principles

1. **Definition versioning** — edit creates new version; activate swaps.  
2. **Cron + calendars** — cron for recurrence; calendars exclude/include days.  
3. **Next-fire precomputation** — store `next_fire_at` index for cheap due scans.  
4. **Misfire policies** — `FIRE_ONCE_NOW`, `SKIP`, `CATCH_UP_N`, `FIRE_ALL_BOUNDED`.  
5. **Overlap policies** — `SKIP`, `QUEUE_ONE`, `ALLOW_PARALLEL`, `FORBID`.  
6. **Blackouts** — global/tenant maintenance windows.  
7. **Jitter** — optional random delay to avoid thundering herds.  
8. **Idempotent fire** — `(schedule_id, planned_fire_at)` unique run.  
9. **Catch-up budget** — max backfill fires per tick.  
10. **Priority** — critical schedules claimed first.  
11. **Tenant fairness** — don’t let one tenant monopolize ticker.  
12. **Feature gates** — schedule active only if flag on.  
13. **Dry-run next fires** — preview without enqueue.  
14. **Dead schedule detection** — never fired / always failing alerts.  
15. **CQRS HTTP** — thin routers; tick engine isolated.  
16. **Packs** — seed system schedules (digest flush, lease sweep, relay tick).  
17. **Fiscal-aware triggers** — optional “first day of FY” via org calendar ref.  
18. **Pause cascades** — pause definition pauses future fires; inflight runs finish.

---

## 4. Core concepts

### 4.1 Schedule kinds

| Kind | Use |
|---|---|
| `CRON` | Standard cron (sec optional) |
| `INTERVAL` | Every N seconds/minutes |
| `ONE_SHOT` | Single `fire_at` |
| `WINDOW` | Fire once randomly/within daily window |
| `CALENDAR_EVENT` | On org fiscal period open/close |
| `DEPENDENT` | After upstream schedule success |

### 4.2 Schedule definition

```text
schedule_key, kind, cron_expr?, timezone,
handler_key, queue_key, payload_template,
misfire_policy, overlap_policy,
calendar_id?, blackout_calendar_id?,
tenant_id?, enabled, next_fire_at, priority
```

### 4.3 Run lifecycle

```text
PLANNED → CLAIMED → ENQUEUED → RUNNING → SUCCEEDED | FAILED | SKIPPED | CANCELLED | MISFIRED
```

### 4.4 Misfire example

Cron every minute; ticker down 10 minutes:

- `SKIP` → next future only  
- `FIRE_ONCE_NOW` → one catch-up then resume  
- `CATCH_UP_N` (N=3) → up to 3 backfill planned fires  

### 4.5 Overlap example

Long report still RUNNING when next fire due:

- `SKIP` → mark new run SKIPPED  
- `QUEUE_ONE` → keep single pending follow-up  
- `FORBID` → alert / incident  

---

## 5. Calendars & blackouts

- **Holiday calendar** — exclude Sundays / Diwali set  
- **Business days only** — combine cron `0 9 * * 1-5` + holiday exclude  
- **Blackout** — “no non-critical schedules 28–31 Mar close”  
- Critical schedules may set `ignore_blackout=true` (permissioned)

---

## 6. Integration patterns

| Consumer | Schedule example |
|---|---|
| p14 | `messaging.sweep.leases` every 30s |
| p13 | `event_bus.relay.tick` every 1s / 5s |
| p15 | `notify.digests.flush` daily 08:00 IST |
| p16 | `cache.warmup.nightly` |
| p07 | `number_series.rollover.check` |
| Domain | `reports.daily_freight` |

Bridge row maps schedule → p14 enqueue payload.

---

## 7. Security

### Permissions

| Code | Use |
|---|---|
| `scheduler.catalog.read` | Read schedules |
| `scheduler.catalog.manage` | Create/edit definitions |
| `scheduler.run` | Manual trigger |
| `scheduler.pause` | Pause/resume |
| `scheduler.admin` | Tickers, blackouts global |
| `scheduler.audit.read` | Audit/runs |
| `scheduler.pack.install` | Packs |
| `scheduler.*` | Wildcard |

### RLS

FORCE RLS on tenant schedules & runs.  
System schedules readable with auth; mutable with manage.

---

## 8. Module layout

```text
platforms/p17_scheduler/
  application/
    services/
      cron_parser.py
      next_fire.py
      tick_engine.py
      misfire.py
      overlap.py
      blackout.py
      enqueue_bridge.py
      catch_up.py
      lock_manager.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… workers/ticker.py
  tests/unit/cron/ misfire/ overlap/ lock/
```

---

## 9. Domain events

| Event | When |
|---|---|
| `scheduler.schedule.activated` / `paused` | Catalog |
| `scheduler.run.planned` / `enqueued` / `succeeded` / `failed` / `skipped` | Runs |
| `scheduler.misfire.detected` | Ops |
| `scheduler.blackout.active` | Ops |
| `scheduler.ticker.stale` | Ops |

---

## 10. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, definitions, permissions |
| P2 | Cron next-fire + tick + enqueue bridge |
| P3 | Run ledger + overlap |
| P4 | Misfire + catch-up |
| P5 | Calendars/blackouts/timezones |
| P6 | Dependencies + one-shots |
| P7 | Fairness + metrics + alerts |
| P8 | Packs (system sweeps) |
| P9 | Registry → **Live** |

---

## 11. Definition of Done (enterprise)

- [ ] Timezone-correct next-fire golden tests (IST/DST edge)  
- [ ] Unique `(schedule_id, planned_fire_at)` prevents double fire  
- [ ] Overlap SKIP works under concurrency  
- [ ] Cluster lock allows only one tick claimer per shard  
- [ ] Misfire CATCH_UP_N respects bound  
- [ ] Blackout suppresses non-critical  
- [ ] Manual run audited  
- [x] Tenant RLS GUCs on calendar/ticker HTTP (`require_scheduler_access`)  
- [x] Calendar + ticker/lock persist on `AsyncSession` (empty catalog is `[]`)  
- [x] No heavy work in ticker  
- [x] No cross-schema FKs  

---

## 12. Anti-patterns

| Don’t | Do |
|---|---|
| Run report generation in ticker | Enqueue p14 job |
| Cron without timezone | Explicit IANA TZ |
| Ignore overlap | Set policy explicitly |
| Infinite catch-up after outage | Bounded CATCH_UP_N |
| One DB poll without lock | Distributed tick lock |
| Server local time assumptions | Store UTC instants + TZ |

---

## 13. Related documents

- Schema: [`SCHEDULER_SCHEMA.md`](SCHEDULER_SCHEMA.md)  
- API: [`SCHEDULER_API.md`](SCHEDULER_API.md)  
- Messaging: [`../14_messaging/MESSAGING_GUIDE.md`](../14_messaging/MESSAGING_GUIDE.md)  
- Organization: [`../02_organization/ORGANIZATION_GUIDE.md`](../02_organization/ORGANIZATION_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
