# JeslotERP Event Bus Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — schema `event_bus`, 61 domain + 3 plumbing = 64 `eb_*` tables; HTTP publish writes `eb_event` + `eb_outbox` same commit  
**Package:** `platforms.p13_event_bus`  
**PostgreSQL schema:** `event_bus`  
**Companion:** [`EVENT_BUS_GUIDE.md`](EVENT_BUS_GUIDE.md) · [`EVENT_BUS_API.md`](EVENT_BUS_API.md)

> Runtime models: `platforms/p13_event_bus/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `event_bus` (never `p13`) |
| Tables | `eb_*` |
| Event types | Dot + version suffix (`.v1`) |
| Envelope | CloudEvents 1.0 + JeslotERP extensions |
| Soft delete | Retire types; retain schemas |
| Cross-schema | No FKs to platform outboxes — source registry |
| RLS | FORCE on tenant delivery/idempotency where tenant-bound |
| Payloads | JSONB; size capped |

---

## 2. Complete table inventory (**61 tables**)

### 2.1 Catalog & schemas (10)

| # | Table | Purpose |
|---|---|---|
| 1 | `eb_event_type` | Event type catalog |
| 2 | `eb_event_type_alias` | Deprecated → canonical |
| 3 | `eb_schema` | Schema versions |
| 4 | `eb_schema_content` | JSON Schema / Avro text |
| 5 | `eb_schema_compat_policy` | Compatibility mode |
| 6 | `eb_schema_activation` | Active schema per type |
| 7 | `eb_domain` | Domain ownership (transport, document, …) |
| 8 | `eb_owner` | Owners |
| 9 | `eb_tag` / join via items | Tags |
| 10 | `eb_event_type_tag` | M2M tags |

### 2.2 Topology (8)

| # | Table | Purpose |
|---|---|---|
| 11 | `eb_topic` | Topics/channels |
| 12 | `eb_topic_type_binding` | Which types route to topic |
| 13 | `eb_subscription` | Subscriptions |
| 14 | `eb_subscription_filter` | Filters |
| 15 | `eb_consumer_group` | Competing consumer groups |
| 16 | `eb_consumer_endpoint` | Push endpoints |
| 17 | `eb_delivery_mode` | Push/pull config |
| 18 | `eb_ordering_policy` | Ordering mode per subscription |

### 2.3 Outbox relay (8)

| # | Table | Purpose |
|---|---|---|
| 19 | `eb_outbox_source` | Registered platform outbox sources |
| 20 | `eb_outbox_source_secret` | secret_ref for DB/creds |
| 21 | `eb_relay_cursor` | Per-source cursor |
| 22 | `eb_relay_batch` | Relay batch runs |
| 23 | `eb_relay_item` | Items in batch |
| 24 | `eb_relay_lag_sample` | Lag metrics |
| 25 | `eb_outbox_contract` | Required outbox column contract |
| 26 | `eb_producer_registration` | Allowed producers |

### 2.4 Event store / index (7)

| # | Table | Purpose |
|---|---|---|
| 27 | `eb_event` | Canonical stored event (retention) |
| 28 | `eb_event_header` | Indexed headers |
| 29 | `eb_event_payload` | Payload (or offload ref) |
| 30 | `eb_event_partition` | Partition assignment |
| 31 | `eb_retention_policy` | Retention rules |
| 32 | `eb_purge_job` | Purge runs |
| 33 | `eb_payload_offload` | Large payload → media_id |

### 2.5 Delivery & ack (8)

| # | Table | Purpose |
|---|---|---|
| 34 | `eb_delivery` | Delivery to subscription |
| 35 | `eb_delivery_attempt` | Attempts |
| 36 | `eb_ack` | Acknowledgements |
| 37 | `eb_nack` | Negative acks |
| 38 | `eb_visibility_lease` | Pull leases |
| 39 | `eb_retry_policy` | Retry configs |
| 40 | `eb_backoff_policy` | Backoff params |
| 41 | `eb_delivery_stats` | Aggregates |

### 2.6 DLQ, poison, replay (7)

| # | Table | Purpose |
|---|---|---|
| 42 | `eb_dlq_entry` | Dead letters |
| 43 | `eb_dlq_reason` | Reason codes |
| 44 | `eb_poison_event` | Poison classifications |
| 45 | `eb_redrive_job` | Redrive runs |
| 46 | `eb_replay_job` | Replay jobs |
| 47 | `eb_replay_cursor` | Replay progress |
| 48 | `eb_replay_filter` | Replay filters |

### 2.7 Idempotency & consumers (5)

| # | Table | Purpose |
|---|---|---|
| 49 | `eb_consumer_idempotency` | Processed event_id per group |
| 50 | `eb_consumer_checkpoint` | Pull checkpoints |
| 51 | `eb_consumer_heartbeat` | Consumer liveness |
| 52 | `eb_handler_registry` | Internal handler keys |
| 53 | `eb_subscription_binding` | Handler ↔ subscription |

### 2.8 Security, governance, packs (8)

| # | Table | Purpose |
|---|---|---|
| 54 | `eb_publish_grant` | Who may publish type/topic |
| 55 | `eb_subscribe_grant` | Who may subscribe |
| 56 | `eb_pii_policy` | PII class per type |
| 57 | `eb_changeset` | Catalog changes |
| 58 | `eb_approval` | Approvals |
| 59 | `eb_package` | Contract packs |
| 60 | `eb_package_item` | Pack items |
| 61 | `eb_webhook_subscription` | Ops webhooks for bus meta |

**Plumbing:** `eb_outbox` (meta), `eb_idempotency_key`, `eb_catalog_audit`

**Implementation total with plumbing: 64 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `eb_schema_compat` | `BACKWARD`, `FORWARD`, `FULL`, `NONE` |
| `eb_relay_status` | `PENDING`, `RELAYING`, `RELAYED`, `FAILED`, `DEAD` |
| `eb_delivery_status` | `PENDING`, `DISPATCHED`, `ACKED`, `NACKED`, `EXPIRED`, `DLQ` |
| `eb_delivery_mode_kind` | `PUSH_HTTP`, `PUSH_WORKER`, `PULL` |
| `eb_ordering_mode` | `NONE`, `KEY`, `TYPE_TENANT` |
| `eb_filter_op` | `TYPE_EQ`, `TYPE_PREFIX`, `TENANT_EQ`, `ATTR_EQ`, `ATTR_IN`, `CE_SUBJECT_PREFIX` |
| `eb_dlq_reason` | `MAX_ATTEMPTS`, `SCHEMA_INVALID`, `HANDLER_ERROR`, `TIMEOUT`, `REJECTED` |
| `eb_pii_class` | `NONE`, `LOW`, `HIGH`, `RESTRICTED` |
| `eb_replay_status` | `PENDING`, `RUNNING`, `COMPLETED`, `CANCELLED`, `FAILED` |

---

## 4. Catalog & schemas (detail)

### 4.1 `eb_event_type`

| Column | Type | Notes |
|---|---|---|
| `type_key` | VARCHAR(200) UNIQUE | `sales.order.cancelled.v1` |
| `domain_id` | UUID | |
| `source_pattern` | VARCHAR(200) | |
| `description` | TEXT NULL | |
| `pii_class` | VARCHAR(20) | |
| `retention_days` | INT | |
| `require_tenant` | BOOLEAN DEFAULT true | |
| `require_partition_key` | BOOLEAN DEFAULT false | |
| `is_active` | BOOLEAN | |
| `owner_platform` | VARCHAR(80) NULL | |

### 4.2 `eb_schema`

| Column | Type | Notes |
|---|---|---|
| `event_type_id` | UUID | |
| `version_number` | INT | |
| `format` | VARCHAR(20) | `JSON_SCHEMA`, `AVRO` |
| `compatibility` | VARCHAR(20) | |
| `checksum` | VARCHAR(64) | |
| `lifecycle` | VARCHAR(20) | DRAFT/PUBLISHED/ACTIVE/RETIRED |
| `activated_at` | TIMESTAMPTZ NULL | |

### 4.3 `eb_schema_content`

| Column | Type | Notes |
|---|---|---|
| `schema_id` | UUID | |
| `content` | TEXT | Schema document |
| `examples` | JSONB NULL | |

---

## 5. Topology

### 5.1 `eb_topic`

| Column | Type | Notes |
|---|---|---|
| `topic_key` | VARCHAR(150) UNIQUE | `jesloterp.document.sample` |
| `name` | VARCHAR(150) | |
| `ordering_default` | VARCHAR(20) | |
| `is_active` | BOOLEAN | |

### 5.2 `eb_subscription`

| Column | Type | Notes |
|---|---|---|
| `subscription_key` | VARCHAR(150) UNIQUE | `notify.document.released` |
| `topic_id` | UUID | |
| `consumer_group_id` | UUID | |
| `delivery_mode` | VARCHAR(20) | |
| `ordering_mode` | VARCHAR(20) | |
| `retry_policy_id` | UUID NULL | |
| `is_active` | BOOLEAN | |
| `tenant_id` | UUID NULL | Null = system-wide |
| `max_concurrency` | INT | |

### 5.3 `eb_subscription_filter`

| Column | Type | Notes |
|---|---|---|
| `subscription_id` | UUID | |
| `op` | VARCHAR(30) | |
| `value_json` | JSONB | |

### 5.4 `eb_consumer_endpoint`

| Column | Type | Notes |
|---|---|---|
| `subscription_id` | UUID | |
| `kind` | VARCHAR(20) | HTTP / WORKER_HANDLER |
| `url` | TEXT NULL | |
| `handler_key` | VARCHAR(100) NULL | |
| `secret_ref_key` | VARCHAR(150) NULL | HMAC for HTTP |
| `timeout_ms` | INT | |

---

## 6. Outbox relay

### 6.1 `eb_outbox_source`

| Column | Type | Notes |
|---|---|---|
| `source_key` | VARCHAR(80) UNIQUE | `p09_document` |
| `platform_code` | VARCHAR(40) | |
| `adapter_kind` | VARCHAR(30) | `PG_TABLE`, `API_PULL` |
| `table_or_endpoint` | VARCHAR(200) | `document.doc_outbox` logical name |
| `claim_batch_size` | INT | |
| `is_active` | BOOLEAN | |
| `poll_interval_ms` | INT | |

> Physical access via secured adapter; **no ORM FK** into other schemas from models.

### 6.2 `eb_relay_cursor`

| Column | Type | Notes |
|---|---|---|
| `source_id` | UUID | |
| `last_outbox_id` | UUID NULL | |
| `last_occurred_at` | TIMESTAMPTZ NULL | |
| `updated_at` | TIMESTAMPTZ | |

### 6.3 `eb_outbox_contract`

Documents required fields platforms must implement (seeded standard). Used by CI validation tooling.

---

## 7. Event store

### 7.1 `eb_event`

| Column | Type | Notes |
|---|---|---|
| `event_id` | UUID PRIMARY | CloudEvents id |
| `type_key` | VARCHAR(200) | |
| `source` | VARCHAR(200) | |
| `subject` | VARCHAR(200) NULL | |
| `time` | TIMESTAMPTZ | |
| `tenant_id` | UUID NULL | Indexed |
| `company_id` | UUID NULL | |
| `correlation_id` | UUID NULL | |
| `causation_id` | UUID NULL | |
| `partition_key` | VARCHAR(200) NULL | |
| `topic_id` | UUID | |
| `schema_id` | UUID | |
| `outbox_source_id` | UUID NULL | |
| `outbox_row_id` | UUID NULL | Stable relay mapping |
| `payload_inline` | JSONB NULL | Small |
| `payload_offload_id` | UUID NULL | Large |
| `checksum` | VARCHAR(64) | |
| `created_at` | TIMESTAMPTZ | |

**Unique:** `(outbox_source_id, outbox_row_id)` when both present.

---

## 8. Delivery

### 8.1 `eb_delivery`

| Column | Type | Notes |
|---|---|---|
| `event_id` | UUID | |
| `subscription_id` | UUID | |
| `status` | VARCHAR(20) | |
| `attempt_count` | INT | |
| `next_attempt_at` | TIMESTAMPTZ NULL | |
| `ordered_seq` | BIGINT NULL | For KEY ordering |
| `locked_by` | VARCHAR(100) NULL | |
| `locked_until` | TIMESTAMPTZ NULL | |

**Unique:** `(event_id, subscription_id)`.

### 8.2 `eb_delivery_attempt`

| Column | Type | Notes |
|---|---|---|
| `delivery_id` | UUID | |
| `attempt_no` | INT | |
| `started_at` / `ended_at` | TIMESTAMPTZ | |
| `http_status` | INT NULL | |
| `error_code` | VARCHAR(50) NULL | |
| `error_detail` | TEXT NULL | |

### 8.3 `eb_consumer_idempotency`

| Column | Type | Notes |
|---|---|---|
| `consumer_group_id` | UUID | |
| `event_id` | UUID | |
| `processed_at` | TIMESTAMPTZ | |
| `result_hash` | VARCHAR(64) NULL | |

**Unique:** `(consumer_group_id, event_id)`.

---

## 9. DLQ & replay

### 9.1 `eb_dlq_entry`

| Column | Type | Notes |
|---|---|---|
| `delivery_id` | UUID | |
| `event_id` | UUID | |
| `subscription_id` | UUID | |
| `reason` | VARCHAR(30) | |
| `enqueued_at` | TIMESTAMPTZ | |
| `redriven_at` | TIMESTAMPTZ NULL | |
| `status` | VARCHAR(20) | OPEN/REDRIVEN/DISCARDED |

### 9.2 `eb_replay_job`

| Column | Type | Notes |
|---|---|---|
| `subscription_id` | UUID | |
| `from_time` / `to_time` | TIMESTAMPTZ | |
| `type_prefix` | VARCHAR(200) NULL | |
| `status` | VARCHAR(20) | |
| `requested_by` | UUID | |
| `events_replayed` | BIGINT | |

---

## 10. Security grants

### 10.1 `eb_publish_grant`

| Column | Type | Notes |
|---|---|---|
| `principal_type` | VARCHAR(20) | SERVICE/ROLE |
| `principal_id` | UUID NULL | |
| `service_key` | VARCHAR(80) NULL | |
| `type_prefix` | VARCHAR(200) | `transport.%` |
| `topic_id` | UUID NULL | |

### 10.2 `eb_subscribe_grant`

Similar for subscription create/bind.

---

## 11. Governance & packs

- Changesets for new types/breaking schema  
- Packages: `core.identity.events@1.0.0`, `document.events@1.0.0`, `process.events@1.0.0`  
- PII policies gate who can read payload in admin APIs  

---

## 12. Plumbing

| Table | Purpose |
|---|---|
| `eb_outbox` | Meta-events (lag, dlq) — isolated channel |
| `eb_idempotency_key` | Admin ops |
| `eb_catalog_audit` | Catalog before/after |

---

## 13. RLS summary

| Class | Policy |
|---|---|
| Catalog/topics system | Read auth; manage permission |
| Tenant subscriptions | FORCE `tenant_id` when set |
| Deliveries/idempotency with tenant | FORCE via event.tenant_id join policy |
| DLQ admin | Permission + tenant scope |

---

## 14. Seed minimum

1. Domains: identity, org, configuration, document, process, feature, media, number_series  
2. Outbox contract standard document row  
3. Retry policy `default_exponential`  
4. Topics per domain  
5. Sample types for document.released, process.instance.completed, feature.kill.engaged  
6. Permissions `event.*`  
7. JSON Schema examples for sample types  

---

## 15. ER overview

```text
domain ── event_type ── schemas ── activation
topic ── type_bindings
subscription ── filters / endpoint / retry / ordering
outbox_source ── cursors / relay_batches
event ── deliveries ── attempts / ack / dlq
consumer_group ── idempotency / checkpoints
replay_job / redrive_job
packages / grants
```

---

## 16. Implementation notes

1. Relay must mark outbox RELAYED only after durable `eb_event` insert.  
2. Push HTTP signs payloads with HMAC secret_ref.  
3. KEY ordering: do not dispatch next seq until prior ACKED/DLQ.  
4. Payload offload to p08 when over threshold.  
5. Split models: `catalog`, `topology`, `relay`, `store`, `delivery`, `dlq`, `security`, `governance`, `plumbing`.
