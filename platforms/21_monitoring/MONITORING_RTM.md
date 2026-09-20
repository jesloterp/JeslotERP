# Monitoring Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p21_monitoring/tests -q --tb=short` → **26 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| MON-G-01 | GUIDE §1 | Observability control plane ≠ logging/audit | `MonitoringCatalogStore` + schema `monitoring` | Implemented | module tables + health | PASS |
| MON-G-02 | GUIDE §2 | Label cardinality; no `user_id`/`email` | `create_metric` / `remote_write` guards | Implemented | `test_metrics_catalog_denies_user_labels`, query cardinality | PASS |
| MON-G-03 | GUIDE §2 | LIVE ≠ READY | `health_live` / `health_ready` + probes | Implemented | `test_status_probes_live_not_ready` | PASS |
| MON-G-04 | GUIDE §2 | No cross-schema FKs | UUID/string refs only on ORM | Implemented | table inventory | PASS |
| MON-G-05 | GUIDE §2 | Secrets via `secret_ref` only | receivers/dashboards/backends strip | Implemented | receivers + dashboards | PASS |
| MON-G-06 | GUIDE §3 | Catalog-first metrics | POST `/metrics` | Implemented | metrics create + 404 | PASS |
| MON-G-07 | GUIDE §3 | SLO-native burn | burn-policy + `/slo/eval` | Implemented | SLO family + invalid burn | PASS |
| MON-G-08 | GUIDE §3 | Probe classes LIVE/READY/DEPENDENCY/SYNTHETIC | probes + synthetics | Implemented | probes + synthetics | PASS |
| MON-G-09 | GUIDE §3 | Alert lifecycle + silences + ticket | ack/resolve/silence | Implemented | alerts + silences | PASS |
| MON-G-10 | GUIDE §3 | Dashboards-as-code versioned | versions + publish | Implemented | dashboards publish + 409 | PASS |
| MON-G-11 | GUIDE §6 | Permissions `monitoring.*` | HTTP gates | Implemented | 403 status | PASS |
| MON-G-12 | GUIDE §8 | Domain events firing/burn/probe/synthetic/silence/dashboard | outbox | Implemented | silence suppress + synthetic fire | PASS |
| MON-G-13 | GUIDE §10 | DoD: LIVE vs READY under dep failure | status DEGRADED | Implemented | ready 503, live 200 | PASS |
| MON-G-14 | GUIDE §10 | DoD: burn eval, high-card reject, silence suppress, synthetic FIRING, no secrets | store + APIs | Implemented | SLO/query/silence/synthetic/dashboard | PASS |
| MON-S-01 | SCHEMA §2 | 60 domain + plumbing = 62 `mon_*` | ORM + outbox + idempotency | Implemented | `test_mon_module_tables_count_62` | PASS |
| MON-S-02 | SCHEMA §1 | Schema `monitoring`, no cross-schema FKs | `MONITORING_SCHEMA` | Implemented | table schema assert | PASS |
| MON-S-03 | SCHEMA §3 | Enums metric/probe/alert/severity/SLO/receiver | `domain/enums.py` | Implemented | API payloads | PASS |
| MON-S-04 | SCHEMA §10 | Seed packs | `seed_defaults` packages | Implemented | packages list | PASS |
| MON-S-05 | SCHEMA §11 | `mon_outbox`, `mon_idempotency_key` | outbox + idempotency models | Implemented | table names | PASS |
| MON-S-06 | SCHEMA §12 | FORCE RLS tenant business/history tables | Alembic `f21b1c2d3e4f` | Implemented | migration present | PASS |
| MON-S-07 | SCHEMA §13 | Seed services/metrics/LIVE-READY/SLO/routes/runbooks/perms | `seed_defaults` + Alembic perms | Implemented | status/metrics/SLO/routes | PASS |
| MON-A-01 | API §4 | MON_* error codes | `domain/exceptions.py` | Implemented | 404/409/422/403/503 families | PASS |
| MON-A-02 | API §5 | Permission codes | `application/permissions/catalog.py` | Implemented | 403 + admin paths | PASS |
| MON-A-03 | API §6 | Status/probes + `/health/live` `/health/ready` | public + health_router | Implemented | status/probes family | PASS |
| MON-A-04 | API §7 | Metrics catalog + scrape + query/query_range | public + TSDB stub | Implemented | metrics + query families | PASS |
| MON-A-05 | API §8 | SLI/SLO/budget/reports/burn/journeys | public + `/slos` (not `/slosos`) | Implemented | SLO family | PASS |
| MON-HYG-021 | AUD-021 | Remove `GET /slosos` typo from API + code | `/slos` only | Implemented | `test_hyg021_slosos_typo` | PASS |
| MON-A-06 | API §9 | Rules, alerts ack/resolve, routes/receivers/escalation, silences | public | Implemented | alerts + routes + silences | PASS |
| MON-A-07 | API §10 | Dashboards versioned publish | public | Implemented | dashboards | PASS |
| MON-A-08 | API §11 | Synthetics run | public | Implemented | synthetics | PASS |
| MON-A-09 | API §12 | Service map + traces | public + rebuild | Implemented | service-map/traces | PASS |
| MON-A-10 | API §13 | Incidents lite | public | Implemented | incidents | PASS |
| MON-A-11 | API §14 | Runbooks + capacity | public | Implemented | runbooks/capacity | PASS |
| MON-A-12 | API §15 | Internal ticks + remote-write | `/internal/v1/monitoring` | Implemented | `test_internal_ticks` | PASS |
| MON-A-13 | API §16–17 | Packages/changesets/backends/health | public | Implemented | packages/backends/health | PASS |
| MON-A-14 | API §20 | Event hooks to outbox | `_emit` | Implemented | firing/silence/synthetic | PASS |
| MON-MOD | registry / brief | ModulePlugin after p20; Alembic f21a/f21b | `MonitoringModule` + main/env | Implemented | load order + deps | PASS |
| MON-SOR-01 | TASK-SOR-019 | Metrics/scrape HTTP → Postgres; empty `[]` | `metrics_repository` + `require_monitoring_access` | Implemented | empty + persist-then-fetch + db-first | PASS |
| MON-SOR-02 | TASK-SOR-019 | TSDB port; no fake Prometheus client | `PendingTsdbAdapter` + factory | Implemented | MEMORY ACTIVE; PROMETHEUS PENDING | PASS |
