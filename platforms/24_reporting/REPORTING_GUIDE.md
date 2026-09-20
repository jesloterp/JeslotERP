# JeslotERP Reporting Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — dataset/report HTTP persists on Postgres; empty list is `[]`. DWH port is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p24_reporting`  
**PostgreSQL schema:** `reporting`  
**Depends on:** `p02_organization`, `p05_metadata`, `p18_search`  
**Integrates with:** `p01_identity`, `p03_configuration`, `p06_localization`, `p08_file_media`, `p12_feature`, `p13_event_bus`, `p14_messaging`, `p15_notification`, `p16_cache`, `p17_scheduler`, `p19_audit`, `p21_monitoring`, `p25_dashboard` (consumes datasets), `p26_licensing` (optional pack entitlements)  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`REPORTING_SCHEMA.md`](REPORTING_SCHEMA.md) · [`REPORTING_API.md`](REPORTING_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-09** | Full enterprise reporting plane: datasets, semantic models, report defs, layouts, parameters, folders/sharing, execution, exports, schedules, row security, snapshots, packs. |
| **1.0 SoR-Live** | **2026-09-12** | TASK-SOR-022: dataset/report HTTP Postgres-first; empty list is `[]`; `require_reporting_access` sets RLS GUCs. DWH stays `PROVIDER_PENDING`. |

---

## 1. Purpose (enterprise)

`p24_reporting` is JeslotERP’s **report definition, dataset, and export control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Crystal / S/4 embedded analytics / BW query definitions** — reusable queries, variants, exports  
- **Microsoft Dynamics 365 / SSRS / Power BI semantic models (ERP-bound)** — datasets, rdl-like defs, subscriptions  
- **Salesforce Reports & Report Types** — folders, filters, columns, scheduled exports (boards stay in p25)  
- **Oracle BI / enterprise financial report catalogs** — governed catalog, security, burst  

It is **not** “CSV download from a list page.” It is the system that makes ERP reporting correct for:

1. **Governed report catalog** — folders, types, ownership, lifecycle  
2. **Datasets & semantic models** — fields, joins, measures (metadata-aligned)  
3. **Report definitions** — columns, groupings, sorts, filters, formulas  
4. **Parameters & variants** — saved run configurations  
5. **Execution engine** — sync/async run with ACL trim  
6. **Exports** — PDF, XLSX, CSV, JSON → `p08` artifacts  
7. **Schedules & subscriptions** — via `p17` clocks → `p14` jobs → notify `p15`  
8. **Row-level security** — tenant/org/branch + dataset policies  
9. **Snapshots** — frozen result sets for audit/period close  
10. **Discoverability** — catalog indexed in `p18` (optional)  

### Owns

| Domain | Examples |
|---|---|
| Catalog / folders | Report tree, favorites |
| Report types | Template shapes |
| Datasets / models | Fields, joins, measures |
| Report defs | Layout, columns, filters |
| Parameters / variants | Run configs |
| Execution | Runs, pages, status |
| Exports | Formats, artifacts |
| Schedules / subscriptions | Burst, email |
| Security | Dataset RLS, share ACLs |
| Snapshots | Period-frozen results |
| Packs | Seeded freight/GST reports |

### Does **not** own

| Concern | Owner |
|---|---|
| Interactive boards / widgets | `p25_dashboard` |
| Operational full-text find | `p18_search` (indexes; not analytical SoR) |
| Byte storage of files | `p08_file_media` |
| Clock / cron | `p17_scheduler` (triggers only) |
| Job workers | `p14_messaging` |
| Entity field truth | Domain + `p05_metadata` |
| Compliance who-changed | `p19_audit` |

### Critical splits

| | **Reporting (p24)** | **Dashboard (p25)** | **Search (p18)** |
|---|---|---|---|
| Purpose | Defined reports & exports | Interactive boards | Find / facets |
| Output | Tabular/PDF/XLSX runs | Widgets / KPIs live | Hit lists |
| Model | Dataset + report def | Widget → dataset/ref | Derived index docs |
| Schedule | First-class subscriptions | Refresh policies lighter | Reindex jobs |

**Rule:** If finance needs “GST outward register for FY, PDF, every Monday,” that is **p24**. If ops wants a live dispatch board, that is **p25** (may bind to a p24 dataset).

---

## 2. Architectural position

```text
Metadata (p05) ──► Dataset / semantic model
Org/Identity    ──► RLS / share ACL
        │
        ▼
 Report definition (layout, filters, params)
        │
   execute (sync | enqueue p14)
        │
   ┌────┴────┐
   │ result  │──► snapshot / page cache (p16)
   │ export  │──► media_id (p08)
   └────┬────┘
        │
   schedule (p17) · notify (p15) · index catalog (p18)
   dashboard widgets (p25) bind dataset_id / report_id
```

**Hard rules**

1. Reports never bypass dataset RLS — fail closed.  
2. Export bytes live in **p08**; reporting stores `media_id` + checksum.  
3. Heavy runs are **async** via p14; sync only for small interactive.  
4. No ad-hoc SQL from clients — only approved dataset queries / compiled plans.  
5. No cross-schema FKs (UUID + gateways).  
6. Definition changes versioned; published vs draft.  
7. Period-close snapshots are immutable.  
8. Catalog mutations audited (p19).

---

## 3. Advanced design principles

1. **Dataset-first** — reports bind to datasets, not raw tables.  
2. **Metadata-aligned fields** — field keys map to p05 where applicable.  
3. **Semantic measures** — SUM/COUNT/AVG with grain rules.  
4. **Layout models** — TABLE, MATRIX, GROUPED, STATEMENT (financial).  
5. **Variant pattern** — same report, many saved parameter sets.  
6. **Compile then run** — validate def → execution plan → run.  
7. **Pagination & streaming** — large exports stream to media.  
8. **Bursting** — one schedule, many recipients with filter slices.  
9. **Share scopes** — PRIVATE, TENANT, ROLE, USER, LINK.  
10. **Feature & license gates** — packs / premium formats via p12/p26.  
11. **Localization** — labels via p06 keys on columns/headers.  
12. **Idempotent schedule runs** — window keys prevent double fire.  
13. **Result TTL** — interactive caches expire; snapshots don’t.  
14. **Query cost budgets** — max rows/time per plan.  
15. **Packs** — seed freight, ledger, GST, fleet packs.  
16. **CQRS** — define/commands vs run/queries.  
17. **Outbox** — `report.completed`, `export.ready`.  
18. **Observability** — run duration/errors → p21.

---

## 4. Core concepts

### 4.1 Dataset

Reusable query surface: sources, joins, fields, measures, default filters, RLS policy ref.

### 4.2 Report definition

Bound to a dataset (or multi-dataset composite): columns, groups, sorts, conditional formats, print options.

### 4.3 Parameter & variant

Runtime inputs (date range, branch, customer). Variant = named saved parameter bag + column visibility.

### 4.4 Execution

`QUEUED → RUNNING → SUCCEEDED | FAILED | CANCELLED` with row counts, duration, plan checksum.

### 4.5 Export

Format + options (landscape, locale, currency) → artifact `media_id`.

### 4.6 Subscription

Schedule binding: cron/event from p17, recipients, burst filters, format, variant.

### 4.7 Snapshot

Immutable frozen result for a business period (e.g. month-end trial balance).

---

## 5. Integration patterns

| Concern | Integration |
|---|---|
| Field catalog | p05 metadata entities/fields |
| Org/branch scope | p02 context + dataset RLS |
| AuthZ | p01 permissions `reporting.*` |
| Labels | p06 message keys |
| Export files | p08 store + signed download |
| Find reports | p18 index report catalog docs |
| Async run/export | p14 job types `reporting.run`, `reporting.export` |
| Clock | p17 job → enqueue p14 |
| Notify ready | p15 templates |
| Cache pages | p16 tags `reporting.result.{run_id}` |
| Audit define/share/export | p19 |
| Live boards | p25 binds `dataset_id` / `report_id` |

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `reporting.catalog.read` | Browse folders/reports |
| `reporting.dataset.read` | Read dataset defs |
| `reporting.dataset.manage` | Manage datasets |
| `reporting.def.manage` | Create/edit report defs |
| `reporting.run` | Execute reports |
| `reporting.export` | Create exports |
| `reporting.schedule.manage` | Subscriptions |
| `reporting.share.manage` | ACL / folders |
| `reporting.snapshot.manage` | Period snapshots |
| `reporting.admin` | Packs, budgets, purge |
| `reporting.*` | Wildcard |

### RLS

FORCE RLS on tenant-owned defs, runs, exports, shares, snapshots.  
System packs readable; editable only by admin.

### Data security

Dataset RLS expressions evaluated with tenant/user/branch claims.  
Export inherits run ACL; link shares time-bound + permission check.

---

## 7. Module layout

```text
platforms/p24_reporting/
  application/
    services/
      dataset_compiler.py
      report_compiler.py
      execution_engine.py
      export_service.py
      subscription_service.py
      rls_evaluator.py
      snapshot_service.py
      catalog_indexer.py
    commands/… queries/…
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… query_adapters/ exporters/
  tests/unit/dataset/ rls/ export/ schedule/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `reporting.dataset.published` | Dataset live |
| `reporting.def.published` / `retired` | Catalog |
| `reporting.run.started` / `completed` / `failed` | Execution |
| `reporting.export.ready` | Artifact available |
| `reporting.subscription.fired` | Schedule |
| `reporting.snapshot.sealed` | Period freeze |
| `reporting.share.changed` | ACL |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, permissions, folders |
| P2 | Datasets + compiler + RLS |
| P3 | Report defs + variants |
| P4 | Sync/async execution |
| P5 | Exports → p08 |
| P6 | Schedules via p17/p14/p15 |
| P7 | Snapshots + packs |
| P8 | Search index + dashboard bindings contract |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

- [ ] Client cannot run arbitrary SQL  
- [ ] RLS fail-closed on every run  
- [ ] Export stores media_id only; download authorized  
- [ ] Schedule double-fire prevented by idempotency window  
- [ ] Draft defs not runnable in prod unless feature allows  
- [ ] Snapshot rows immutable after seal  
- [ ] Large runs async; sync guarded by row/time budget  
- [ ] No cross-schema FKs  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Embed SQL in frontend | Dataset + compiled plan |
| Store XLSX in Postgres | p08 media |
| Duplicate dashboard engine here | p25 for boards |
| Use search index as financial SoR | Datasets against operational/warehouse read models |
| Email full PII without ACL | Subscription respects share + RLS |
| Mutate sealed snapshots | New snapshot version |

---

## 12. Related documents

- Schema: [`REPORTING_SCHEMA.md`](REPORTING_SCHEMA.md)  
- API: [`REPORTING_API.md`](REPORTING_API.md)  
- Metadata: [`../05_metadata/METADATA_GUIDE.md`](../05_metadata/METADATA_GUIDE.md)  
- Search: [`../18_search/SEARCH_GUIDE.md`](../18_search/SEARCH_GUIDE.md)  
- Dashboard: [`../25_dashboard/DASHBOARD_GUIDE.md`](../25_dashboard/DASHBOARD_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
