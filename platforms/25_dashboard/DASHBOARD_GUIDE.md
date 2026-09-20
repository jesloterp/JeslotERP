# JeslotERP Dashboard Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — folder/board/widget HTTP persists on Postgres; empty list is `[]`. Widget data/cache stays memory. Not Production.  
**Package:** `platforms.p25_dashboard`  
**PostgreSQL schema:** `dashboard`  
**Depends on:** `p03_configuration`, `p12_feature`, `p24_reporting`  
**Integrates with:** `p01_identity`, `p02_organization`, `p06_localization`, `p08_file_media`, `p13_event_bus`, `p14_messaging`, `p15_notification`, `p16_cache`, `p17_scheduler`, `p18_search`, `p19_audit`, `p21_monitoring` (ops SLOs ≠ ERP boards), `p26_licensing`  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`DASHBOARD_SCHEMA.md`](DASHBOARD_SCHEMA.md) · [`DASHBOARD_API.md`](DASHBOARD_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise dashboard plane: boards, layouts, widgets, bindings to p24 datasets, filters, personalization, refresh, sharing, home assignment, drill-through, threshold alerts, packs. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-022: folder/board/widget HTTP Postgres-first; empty list is `[]`; `require_dashboard_access` sets RLS GUCs. |

---

## 1. Purpose (enterprise)

`p25_dashboard` is JeslotERP’s **interactive board & widget control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Fiori Overview Pages / Analytical List Pages / SAC stories (board layer)** — tiles, KPIs, filters  
- **Microsoft Dynamics 365 dashboards / Power BI embedded tiles (ERP shell)** — role home, personal vs system  
- **Salesforce Home / Lightning Dashboards** — components, running user, folders  
- **Oracle / industrial ERP cockpits** — dense ops boards, not marketing widgets  

It is **not** “a React chart on the homepage.” It is the system that makes ERP dashboards correct for:

1. **Governed board catalog** — system, tenant, personal boards  
2. **Layouts** — grid, sections, responsive breakpoints  
3. **Widgets** — KPI, chart, table, list, filter, embed report, markdown, iframe (policy)  
4. **Data bindings** — primarily **p24 datasets** / report slices (no ad-hoc SQL)  
5. **Global & widget filters** — date, branch, partner, synced filter bar  
6. **Refresh policies** — live, interval, manual, event-driven  
7. **Personalization** — user layout overrides within policy  
8. **Sharing & roles** — default home board per role  
9. **Drill-through** — widget → report run / entity route  
10. **Threshold alerts** — widget breach → p15 (optional)  

### Owns

| Domain | Examples |
|---|---|
| Boards / pages | Home, Dispatch, Finance |
| Layouts / sections | Grid placements |
| Widgets | KPI, chart, table… |
| Bindings | Dataset/report refs |
| Filters | Board + widget filters |
| Personalization | User overrides |
| Refresh | Policies, cache keys |
| Sharing / home | Role defaults |
| Drill / actions | Navigation contracts |
| Threshold alerts | Alert rules |
| Packs | Ops/finance/fleet boards |

### Does **not** own

| Concern | Owner |
|---|---|
| Datasets, report defs, PDF/XLSX exports | `p24_reporting` |
| Ops SLO burn / LIVE probes | `p21_monitoring` |
| Feature flag evaluation | `p12_feature` |
| Tenant settings storage | `p03_configuration` |
| Chart rendering library | Frontend |
| Full scheduled report subscriptions | `p24` (+ p17) |

### Critical splits

| | **Dashboard (p25)** | **Reporting (p24)** | **Monitoring (p21)** |
|---|---|---|---|
| Purpose | Interactive ops/exec boards | Defined reports & exports | Platform health/SLOs |
| Output | Widgets / tiles | Tabular/PDF runs | Metrics/alerts |
| Data | Binds datasets | Owns datasets | Telemetry series |
| Schedule | Refresh cadence | Subscriptions/exports | Scrape/eval |

**Rule:** “GST register every Monday as PDF” → **p24**. “Today’s freight KPIs on Dispatch home” → **p25** (widgets bound to p24 datasets).

---

## 2. Architectural position

```text
p24 dataset / report binding contract
            │
            ▼
   Board + layout + widgets
            │
   filter bar → query params
            │
   refresh (cache p16 / async p14)
            │
   ┌────────┴────────┐
   │ widget payload  │──► UI
   │ threshold check │──► p15 (optional)
   └────────┬────────┘
            │
   drill → report run / deep link
   home assignment ← roles (p01)
   feature packs ← p12 / p26
```

**Hard rules**

1. Widgets **must not** embed raw SQL — only approved dataset/report bindings.  
2. RLS is inherited from **p24 dataset RLS** + board share ACL — fail closed.  
3. Heavy widget queries use reporting run/preview budgets; cache via p16.  
4. Personalization cannot escalate privileges beyond board share.  
5. No cross-schema FKs (UUID + gateways).  
6. System boards editable only by admin; tenants clone to customize.  
7. Board publish is versioned (draft vs live).  
8. Definition changes audited (p19).

---

## 3. Advanced design principles

1. **Board-first** — widgets live on boards, not free-floating.  
2. **Layout as data** — grid coords, span, z-order, breakpoints.  
3. **Widget type registry** — capabilities & config schema.  
4. **Binding contracts** — stable field/measure keys from p24.  
5. **Filter inheritance** — board filters push to widgets unless overridden.  
6. **Running user security** — always evaluate as viewer (not as the author).  
7. **Density modes** — compact ERP vs comfortable (theme hint).  
8. **Refresh classes** — REALTIME | INTERVAL | MANUAL | ON_OPEN.  
9. **Stale-while-revalidate** — show cache, refresh background.  
10. **Drill contracts** — typed targets (REPORT, ENTITY_ROUTE, BOARD).  
11. **Role home** — one default board per role/tenant.  
12. **Clone & fork** — system pack → tenant board.  
13. **Feature-gated widgets** — premium charts via p12.  
14. **Localization** — titles via p06 keys.  
15. **CQRS** — define boards vs query widget data.  
16. **Outbox** — `dashboard.published`, `widget.alert.fired`.  
17. **Search** — catalog boards in p18 optional.  
18. **Observability** — widget error rates → p21 (product metrics).

---

## 4. Core concepts

### 4.1 Board

Named interactive page: lifecycle, owner, folder, layout kind, default filters.

### 4.2 Layout / section

Grid definition: columns, row height, sections (e.g. “Today”, “Exceptions”).

### 4.3 Widget

Typed component instance: position, config, binding, local filters, refresh.

### 4.4 Binding

Reference to `dataset_id` / `report_id` + measure/field projection + aggregation hint.

### 4.5 Filter bar

Board-level parameters (date range, branch) synced to widgets.

### 4.6 Personalization

Per-user overrides: hide widget, rearrange (if allowed), saved filter presets.

### 4.7 Home assignment

Maps role (or user) → board_id for shell landing.

### 4.8 Threshold alert

Widget rule: measure op value → notify channel / in-app badge.

---

## 5. Integration patterns

| Concern | Integration |
|---|---|
| Data | p24 binding-contract + preview/run APIs |
| Settings | p03 defaults (home board prefs keys) |
| Features | p12 gate packs/widget types |
| AuthZ | p01 `dashboard.*` + share ACL |
| Labels | p06 |
| Cache | p16 `dashboard.widget.{board}.{widget}.{hash}` |
| Async refresh | p14 job `dashboard.refresh` |
| Clock refresh | p17 optional for warm cache |
| Alerts | p15 |
| Audit | p19 on publish/share |
| Ops vs board | p21 remains infrastructure monitoring |

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `dashboard.catalog.read` | Browse boards |
| `dashboard.board.manage` | Create/edit boards |
| `dashboard.widget.manage` | Edit widgets/layouts |
| `dashboard.personalize` | User overrides |
| `dashboard.share.manage` | ACL / home assign |
| `dashboard.publish` | Publish live |
| `dashboard.alert.manage` | Threshold rules |
| `dashboard.admin` | Packs, system boards |
| `dashboard.*` | Wildcard |

### RLS

FORCE RLS on tenant boards, personalizations, shares, alert subscriptions.  
System pack boards readable when published; clone creates tenant copy.

### Data security

Widget data fetch always as **running user** with dataset RLS.  
Share `VIEW` does not grant dataset manage.

---

## 7. Module layout

```text
platforms/p25_dashboard/
  application/
    services/
      board_service.py
      layout_engine.py
      widget_registry.py
      binding_resolver.py
      filter_merger.py
      refresh_service.py
      personalization.py
      home_assignment.py
      threshold_alerts.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… reporting_gateway/ cache/
  tests/unit/layout/ filter/ binding/ personalize/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `dashboard.board.published` / `retired` | Catalog |
| `dashboard.share.changed` | ACL |
| `dashboard.home.assigned` | Role home |
| `dashboard.widget.refreshed` | Cache warm (sampled) |
| `dashboard.alert.fired` / `cleared` | Thresholds |
| `dashboard.personalization.saved` | User |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, permissions, board CRUD |
| P2 | Layout + widget registry |
| P3 | p24 binding + filter bar |
| P4 | Refresh + cache |
| P5 | Share + role home |
| P6 | Personalization |
| P7 | Drill + threshold alerts |
| P8 | Packs (ops/finance) |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [ ] No widget path executes client SQL  
- [ ] Dataset RLS enforced on every widget data call  
- [ ] Personalization cannot reveal unauthorized widgets’ data  
- [ ] Published board versions immutable; edits create draft  
- [ ] Role home resolves deterministically  
- [ ] Cache keys include tenant, user claims hash, filters  
- [ ] Feature-gated widget types denied when flag off  
- [ ] No cross-schema FKs  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Duplicate dataset engine in p25 | Bind p24 |
| Store export PDFs as “dashboards” | p24 exports |
| Put LIVE/READY probes on boards | p21 |
| Global admin bypass of RLS for “pretty demos” | Fail closed |
| Infinite auto-refresh without budget | Refresh policy + cache |
| Hardcode home route per role in frontend only | Home assignment API |

---

## 12. Related documents

- Schema: [`DASHBOARD_SCHEMA.md`](DASHBOARD_SCHEMA.md)  
- API: [`DASHBOARD_API.md`](DASHBOARD_API.md)  
- Reporting: [`../24_reporting/REPORTING_GUIDE.md`](../24_reporting/REPORTING_GUIDE.md)  
- Configuration: [`../03_configuration/CONFIGURATION_GUIDE.md`](../03_configuration/CONFIGURATION_GUIDE.md)  
- Feature: [`../12_feature/FEATURE_GUIDE.md`](../12_feature/FEATURE_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
