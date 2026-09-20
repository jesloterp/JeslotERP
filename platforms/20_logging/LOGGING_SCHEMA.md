# JeslotERP Logging Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `log_hot_record` + catalog tables are the HTTP ledger. Empty list is `[]`. Not Production.  
**Package:** `platforms.p20_logging`  
**PostgreSQL schema:** `logging`  
**Companion:** [`LOGGING_GUIDE.md`](LOGGING_GUIDE.md) · [`LOGGING_API.md`](LOGGING_API.md)

> Runtime models: `platforms/p20_logging/infrastructure/persistence/models/`.  
> **Note:** High-volume log bodies primarily live in sinks / hot store partitions; Postgres holds control plane + optional hot index meta.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `logging` (never `p20`) |
| Tables | `log_*` |
| Soft delete | TTL purge; no user deletes of others’ forensic windows without admin |
| Cross-schema | UUID attrs only |
| RLS | FORCE on tenant hot records |
| Secrets | Sink credentials via configuration `secret_ref` |

---

## 2. Complete table inventory (**58 tables**)

### 2.1 Schema & catalog (9)

| # | Table | Purpose |
|---|---|---|
| 1 | `log_field_catalog` | Allowed/required fields |
| 2 | `log_level` | Level catalog |
| 3 | `log_category` | Categories |
| 4 | `log_service` | Services/components |
| 5 | `log_environment` | Env keys |
| 6 | `log_event_name` | Optional event names |
| 7 | `log_resource_attr` | Resource attribute defs |
| 8 | `log_schema_version` | Envelope versions |
| 9 | `log_feature_binding` | Feature gates |

### 2.2 Levels & overrides (5)

| # | Table | Purpose |
|---|---|---|
| 10 | `log_level_policy` | Default levels |
| 11 | `log_level_override` | Time-boxed overrides |
| 12 | `log_level_override_scope` | service/tenant/env scopes |
| 13 | `log_level_audit` | Override audit |
| 14 | `log_dynamic_config` | Hot reload config blob meta |

### 2.3 Pipelines & processors (8)

| # | Table | Purpose |
|---|---|---|
| 15 | `log_pipeline` | Pipelines |
| 16 | `log_processor` | Processor defs |
| 17 | `log_pipeline_step` | Ordered steps |
| 18 | `log_scrub_rule` | PII/scrub rules |
| 19 | `log_enrich_rule` | Enrichment |
| 20 | `log_sample_policy` | Sampling |
| 21 | `log_rate_limit_policy` | Rate limits |
| 22 | `log_drop_policy` | Drop rules under pressure |

### 2.4 Sinks & shippers (8)

| # | Table | Purpose |
|---|---|---|
| 23 | `log_sink` | Sink registry |
| 24 | `log_sink_secret` | secret_ref |
| 25 | `log_sink_route` | Pipeline → sink filters |
| 26 | `log_sink_health` | Health samples |
| 27 | `log_shipper` | Agent/shipper instances |
| 28 | `log_shipper_manifest` | Desired config |
| 29 | `log_shipper_heartbeat` | Heartbeats |
| 30 | `log_export_batch` | Export batches meta |

### 2.5 Hot store (7)

| # | Table | Purpose |
|---|---|---|
| 31 | `log_hot_record` | Recent structured logs (partitioned) |
| 32 | `log_hot_attr` | Indexed attrs (optional EAV) |
| 33 | `log_hot_partition` | Partition registry |
| 34 | `log_hot_retention` | Hot TTL |
| 35 | `log_hot_purge_job` | Purge jobs |
| 36 | `log_hot_index_hint` | Index hints |
| 37 | `log_tail_cursor` | Live tail cursors |

### 2.6 Fingerprints & signals (6)

| # | Table | Purpose |
|---|---|---|
| 38 | `log_fingerprint` | Error fingerprints |
| 39 | `log_fingerprint_hit` | Occurrences |
| 40 | `log_fingerprint_owner` | Assignment to team |
| 41 | `log_signal_rule` | Promote to p21/notify |
| 42 | `log_signal_event` | Fired signals |
| 43 | `log_storm_detector` | Burst detection |

### 2.7 Query access & analytics (7)

| # | Table | Purpose |
|---|---|---|
| 44 | `log_saved_query` | Saved queries |
| 45 | `log_query_access` | Who queried |
| 46 | `log_query_stats` | Query metrics |
| 47 | `log_volume_stats` | Ingest volume |
| 48 | `log_drop_stats` | Dropped counts |
| 49 | `log_service_slo_hint` | Error-rate hints |
| 50 | `log_tenant_quota` | Tenant log quotas |

### 2.8 Governance & packs (8)

| # | Table | Purpose |
|---|---|---|
| 51 | `log_changeset` | Pipeline changes |
| 52 | `log_approval` | Approvals |
| 53 | `log_package` | Packs |
| 54 | `log_package_item` | Items |
| 55 | `log_simulator_run` | Scrub/sample sims |
| 56 | `log_catalog_audit` | Audit |
| 57 | `log_sdk_client` | Registered SDKs |
| 58 | `log_ingest_token` | Ingest token meta (hash) |

**Plumbing:** `log_outbox`, `log_idempotency_key`

**Implementation total with plumbing: 60 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `log_level_code` | `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` |
| `log_sink_kind` | `STDOUT`, `OTLP`, `ELASTIC`, `LOKI`, `CLOUDWATCH`, `FILE`, `SIEM_HTTP` |
| `log_processor_kind` | `SCRUB`, `ENRICH`, `SAMPLE`, `RATE_LIMIT`, `FILTER`, `FINGERPRINT`, `ROUTE` |
| `log_sample_mode` | `PROBABILISTIC`, `RATE`, `ERROR_ALWAYS`, `TAIL_BASED` |
| `log_override_status` | `ACTIVE`, `EXPIRED`, `CANCELLED` |
| `log_shipper_status` | `ONLINE`, `STALE`, `OFFLINE` |

---

## 4. Field catalog

### 4.1 `log_field_catalog`

| Column | Type | Notes |
|---|---|---|
| `field_key` | VARCHAR(80) UNIQUE | `tenant_id`, `request_id` |
| `data_type` | VARCHAR(20) | |
| `required` | BOOLEAN | |
| `pii_class` | VARCHAR(20) | |
| `indexed_hot` | BOOLEAN | |
| `allowed_in_sink` | BOOLEAN | |

Required core: `timestamp`, `level`, `message`, `service`, `env`.

---

## 5. Services & overrides

### 5.1 `log_service`

| Column | Type | Notes |
|---|---|---|
| `service_key` | VARCHAR(80) UNIQUE | `api`, `worker-media` |
| `owner_team` | VARCHAR(100) NULL | |
| `default_level` | VARCHAR(10) | |
| `platform_code` | VARCHAR(40) NULL | |

### 5.2 `log_level_override`

| Column | Type | Notes |
|---|---|---|
| `level` | VARCHAR(10) | |
| `status` | VARCHAR(20) | |
| `expires_at` | TIMESTAMPTZ | |
| `reason` | TEXT | |
| `created_by` | UUID | |

Scopes in `log_level_override_scope`: `env`, `service_key`, `tenant_id`, `component`.

---

## 6. Pipelines

### 6.1 `log_pipeline_step`

| Column | Type | Notes |
|---|---|---|
| `pipeline_id` | UUID | |
| `position` | INT | |
| `processor_id` | UUID | |
| `config` | JSONB | |
| `is_enabled` | BOOLEAN | |

### 6.2 `log_scrub_rule`

| Column | Type | Notes |
|---|---|---|
| `rule_key` | VARCHAR(50) | |
| `pattern` | TEXT | |
| `replacement` | VARCHAR(50) | `[REDACTED]` |
| `fields` | JSONB NULL | If null, all string fields |

### 6.3 `log_sample_policy`

| Column | Type | Notes |
|---|---|---|
| `mode` | VARCHAR(20) | |
| `ratio` | NUMERIC(5,4) NULL | |
| `per_second` | INT NULL | |
| `always_levels` | JSONB | `["ERROR","FATAL"]` |

---

## 7. Sinks & shippers

### 7.1 `log_sink`

| Column | Type | Notes |
|---|---|---|
| `sink_key` | VARCHAR(50) UNIQUE | |
| `kind` | VARCHAR(30) | |
| `endpoint` | TEXT NULL | |
| `secret_ref_key` | VARCHAR(150) NULL | |
| `is_active` | BOOLEAN | |
| `config` | JSONB | non-secret |

### 7.2 `log_shipper_manifest`

Desired agent config version + checksum; shippers poll/reconcile.

---

## 8. Hot store

### 8.1 `log_hot_record`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID/BIGSERIAL | |
| `ts` | TIMESTAMPTZ | Partition key |
| `level` | VARCHAR(10) | |
| `service` | VARCHAR(80) | |
| `env` | VARCHAR(30) | |
| `tenant_id` | UUID NULL | RLS |
| `request_id` | UUID NULL | |
| `trace_id` | VARCHAR(64) NULL | |
| `category` | VARCHAR(40) NULL | |
| `message` | TEXT | |
| `attrs` | JSONB | |
| `fingerprint` | VARCHAR(64) NULL | |
| `raw_size` | INT | |

Partition by day/hour; purge by `log_hot_retention`.

---

## 9. Fingerprints

### 9.1 `log_fingerprint`

| Column | Type | Notes |
|---|---|---|
| `fingerprint` | VARCHAR(64) UNIQUE | |
| `service` | VARCHAR(80) | |
| `error_type` | VARCHAR(150) NULL | |
| `title` | VARCHAR(200) | |
| `first_seen_at` | TIMESTAMPTZ | |
| `last_seen_at` | TIMESTAMPTZ | |
| `hit_count` | BIGINT | |
| `status` | VARCHAR(20) | OPEN/ACK/RESOLVED |

---

## 10. Access & quotas

### 10.1 `log_query_access`

| Column | Type | Notes |
|---|---|---|
| `actor_id` | UUID | |
| `query_hash` | VARCHAR(64) | |
| `result_count` | INT | |
| `occurred_at` | TIMESTAMPTZ | |

### 10.2 `log_tenant_quota`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `max_ingest_per_min` | INT | |
| `max_hot_query_per_min` | INT | |

---

## 11. Governance & packs

Seed packs:

- `logging.pipeline.prod.default@1.0.0` — scrub+sample+otlp  
- `logging.scrub.india.pii@1.0.0` — PAN/GSTIN/phone  
- `logging.services.core@1.0.0` — api/workers catalog  

---

## 12. Plumbing

| Table | Purpose |
|---|---|
| `log_outbox` | Meta events |
| `log_idempotency_key` | Admin APIs |

---

## 13. RLS summary

| Class | Policy |
|---|---|
| Catalog/pipelines | Read auth; manage permission |
| Hot records with tenant | FORCE `tenant_id` |
| Overrides | Admin + audit |
| Sink secrets | Admin only |

---

## 14. Seed minimum

1. Field catalog + schema version `1.0`  
2. Levels/categories  
3. Services: `api`, `worker-default`, `scheduler-ticker`  
4. Pipeline `default` with scrub + error-always sample  
5. Sink `stdout` + optional `otlp_primary`  
6. Hot retention 72h  
7. Permissions `logging.*`  

---

## 15. ER overview

```text
field_catalog / services / levels
pipeline ── steps ── processors (scrub/sample/…)
       ── sink_routes ── sinks / shippers
hot_record (partitioned) ── fingerprints
overrides / quotas / query_access
packages
```

---

## 16. Implementation notes

1. Ingest path should avoid synchronous heavy ES writes in API process — buffer/queue.  
2. Hot store optional if external Loki/ES is primary query; still keep control plane in PG.  
3. Fingerprint computation must be stable across deploys (normalize paths/line numbers carefully).  
4. Split models: `catalog`, `pipeline`, `sink`, `hot`, `fingerprint`, `access`, `governance`, `plumbing`.
