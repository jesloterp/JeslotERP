# JeslotERP Integration Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `integration_outbound_message` + `integration_delivery_attempt` are the HTTP delivery ledger. Adapter kind lives in `metadata_fields` (no new column). Not Production.  
**Package:** `platforms.p23_integration`  
**PostgreSQL schema:** `integration`  
**Companion:** [`INTEGRATION_GUIDE.md`](INTEGRATION_GUIDE.md) · [`INTEGRATION_API.md`](INTEGRATION_API.md)

> Runtime models: `platforms/p23_integration/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `integration` (never `p23`) |
| Tables | `integration_*` |
| Soft delete | Deactivate connections; retain delivery history |
| Cross-schema | UUID refs (tenant, partner, media, schedule, job) |
| RLS | FORCE on tenant connections, deliveries, receptions, DLQ, profiles |
| Secrets | `secret_ref` / `vault_path` only — never plaintext |

---

## 2. Complete table inventory (**63 tables**)

### 2.1 Connector catalog (7)

| # | Table | Purpose |
|---|---|---|
| 1 | `integration_connector` | Connector types |
| 2 | `integration_connector_capability` | Caps (oauth, batch, …) |
| 3 | `integration_connector_auth_mode` | Allowed auth modes |
| 4 | `integration_connector_config_schema` | JSON schema for config |
| 5 | `integration_adapter_version` | Adapter impl versions |
| 6 | `integration_connector_pack_link` | Pack membership |
| 7 | `integration_feature_binding` | Feature gates |

### 2.2 Connections & secrets (9)

| # | Table | Purpose |
|---|---|---|
| 8 | `integration_connection` | Connection instances |
| 9 | `integration_connection_endpoint` | URLs / hosts |
| 10 | `integration_connection_auth` | Auth mode + secret refs |
| 11 | `integration_connection_tls` | TLS / mTLS refs |
| 12 | `integration_secret_ref` | Secret pointer registry |
| 13 | `integration_connection_header` | Static headers |
| 14 | `integration_connection_env` | SANDBOX/LIVE meta |
| 15 | `integration_connection_health` | Last probe |
| 16 | `integration_connection_rotation` | Credential rotation jobs |

### 2.3 Mappings & transforms (8)

| # | Table | Purpose |
|---|---|---|
| 17 | `integration_mapping` | Mapping headers |
| 18 | `integration_mapping_version` | Versions |
| 19 | `integration_mapping_field` | Field maps |
| 20 | `integration_mapping_constant` | Constants |
| 21 | `integration_mapping_script` | Sandbox scripts |
| 22 | `integration_mapping_test_case` | Fixture tests |
| 23 | `integration_canonical_schema` | Canonical message schemas |
| 24 | `integration_external_schema` | External schemas |

### 2.4 Pipelines / flows (9)

| # | Table | Purpose |
|---|---|---|
| 25 | `integration_pipeline` | Pipelines |
| 26 | `integration_pipeline_version` | Versions |
| 27 | `integration_pipeline_step` | Steps |
| 28 | `integration_pipeline_trigger` | Event/schedule/manual |
| 29 | `integration_pipeline_binding` | Connection + mapping binds |
| 30 | `integration_pipeline_branch` | Conditional branches |
| 31 | `integration_pipeline_error_policy` | On-error behavior |
| 32 | `integration_pipeline_publish` | Publish records |
| 33 | `integration_changeset` | Changesets |

### 2.5 Outbound delivery (8)

| # | Table | Purpose |
|---|---|---|
| 34 | `integration_outbound_message` | Logical outbound msgs |
| 35 | `integration_delivery_attempt` | Attempts |
| 36 | `integration_retry_policy` | Retry policies |
| 37 | `integration_rate_limit_policy` | Outbound rate limits |
| 38 | `integration_circuit_breaker` | Breaker state |
| 39 | `integration_bulkhead` | Concurrency caps |
| 40 | `integration_outbound_payload` | Payload meta / media_id |
| 41 | `integration_outbound_response` | Response meta / media_id |

### 2.6 Inbound webhooks (8)

| # | Table | Purpose |
|---|---|---|
| 42 | `integration_webhook_endpoint` | Inbound endpoints |
| 43 | `integration_webhook_secret` | Verify secret refs |
| 44 | `integration_webhook_subscription` | External event filters |
| 45 | `integration_inbound_reception` | Received hits |
| 46 | `integration_inbound_verification` | Verify results |
| 47 | `integration_inbound_process` | Process status |
| 48 | `integration_inbound_idempotency` | Idempotency keys |
| 49 | `integration_inbound_payload` | Payload meta / media_id |

### 2.7 DLQ, reconcile, partners (9)

| # | Table | Purpose |
|---|---|---|
| 50 | `integration_dlq_item` | Dead letters |
| 51 | `integration_dlq_replay` | Replay audits |
| 52 | `integration_reconcile_object` | External id map |
| 53 | `integration_sync_state` | Cursor/watermark |
| 54 | `integration_partner_profile` | Partner profiles |
| 55 | `integration_partner_pipeline` | Allowed pipelines |
| 56 | `integration_partner_connection` | Allowed connections |
| 57 | `integration_sla_policy` | SLA targets |
| 58 | `integration_alert_rule` | Ops alert rules |

### 2.8 Governance & analytics (5)

| # | Table | Purpose |
|---|---|---|
| 59 | `integration_package` | Packs |
| 60 | `integration_package_item` | Items |
| 61 | `integration_usage_rollup` | Call volume |
| 62 | `integration_error_rollup` | Error classes |
| 63 | `integration_approval` | Publish approvals |

**Plumbing:** `integration_outbox`, `integration_idempotency_key`

**Implementation total with plumbing: 65 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `integration_connector_kind` | `REST_JSON`, `SOAP`, `SFTP`, `WEBHOOK_IN`, `EDI`, `GRAPHQL`, `CUSTOM` |
| `integration_auth_mode` | `NONE`, `API_KEY`, `BASIC`, `OAUTH2_CC`, `OAUTH2_ROPC`, `MTLS`, `HMAC`, `SIGNED_JWT` |
| `integration_env` | `SANDBOX`, `LIVE` |
| `integration_direction` | `OUTBOUND`, `INBOUND`, `BIDIRECTIONAL` |
| `integration_step_type` | `MAP`, `CALL`, `BRANCH`, `EMIT_EVENT`, `STORE`, `WAIT`, `NOTIFY_OPS` |
| `integration_delivery_status` | `PENDING`, `RUNNING`, `SUCCEEDED`, `RETRYING`, `FAILED`, `DLQ`, `CANCELLED` |
| `integration_error_class` | `RETRYABLE`, `AUTH`, `VALIDATION`, `POISON`, `TIMEOUT`, `RATE_LIMIT`, `DEPENDENCY` |
| `integration_breaker_state` | `CLOSED`, `OPEN`, `HALF_OPEN` |
| `integration_lifecycle` | `DRAFT`, `PUBLISHED`, `DEPRECATED`, `RETIRED` |

---

## 4. Connector & connection detail

### 4.1 `integration_connector`

| Column | Type | Notes |
|---|---|---|
| `connector_key` | VARCHAR(80) UNIQUE | `rest.json.v1` |
| `name` | VARCHAR(150) | |
| `kind` | VARCHAR(30) | |
| `is_system` | BOOLEAN | |
| `adapter_module` | VARCHAR(200) | Python entry |

### 4.2 `integration_connection`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | RLS |
| `connector_id` | UUID | |
| `connection_key` | VARCHAR(100) | |
| `name` | VARCHAR(150) | |
| `env` | VARCHAR(20) | |
| `status` | VARCHAR(20) | `ACTIVE`, `DISABLED`, `ERROR` |
| `partner_id` | UUID NULL | Soft p04 |
| `config` | JSONB | Non-secret config |

### 4.3 `integration_connection_auth`

| Column | Type | Notes |
|---|---|---|
| `connection_id` | UUID | |
| `auth_mode` | VARCHAR(30) | |
| `api_key_secret_ref` | VARCHAR(200) NULL | |
| `oauth_client_secret_ref` | VARCHAR(200) NULL | |
| `oauth_token_url` | VARCHAR(500) NULL | |
| `username_secret_ref` | VARCHAR(200) NULL | |
| `password_secret_ref` | VARCHAR(200) NULL | |
| `hmac_secret_ref` | VARCHAR(200) NULL | |

### 4.4 `integration_secret_ref`

| Column | Type | Notes |
|---|---|---|
| `ref_key` | VARCHAR(120) | Tenant-scoped unique |
| `provider` | VARCHAR(40) | `vault`, `config`, `k8s` |
| `path` | VARCHAR(300) | |
| `version` | VARCHAR(40) NULL | |
| `rotated_at` | TIMESTAMPTZ NULL | |

---

## 5. Mapping detail

### 5.1 `integration_mapping_version`

| Column | Type | Notes |
|---|---|---|
| `mapping_id` | UUID | |
| `version` | INT | |
| `direction` | VARCHAR(20) | |
| `checksum` | VARCHAR(64) | |
| `lifecycle` | VARCHAR(20) | |

### 5.2 `integration_mapping_field`

| Column | Type | Notes |
|---|---|---|
| `mapping_version_id` | UUID | |
| `source_path` | VARCHAR(300) | |
| `target_path` | VARCHAR(300) | |
| `transform` | VARCHAR(80) NULL | `UPPER`, `DATE_ISO`, `CUSTOM` |
| `required` | BOOLEAN | |
| `default_json` | JSONB NULL | |

Scripts in `integration_mapping_script` are sandboxed; no network/IO.

---

## 6. Pipeline detail

### 6.1 `integration_pipeline`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | NULL = system pack |
| `pipeline_key` | VARCHAR(100) | |
| `name` | VARCHAR(150) | |
| `direction` | VARCHAR(20) | |
| `lifecycle` | VARCHAR(20) | |
| `published_version_id` | UUID NULL | |

### 6.2 `integration_pipeline_step`

| Column | Type | Notes |
|---|---|---|
| `pipeline_version_id` | UUID | |
| `step_key` | VARCHAR(80) | |
| `ordinal` | INT | |
| `step_type` | VARCHAR(30) | |
| `config` | JSONB | |
| `on_error` | VARCHAR(30) | `FAIL`, `CONTINUE`, `DLQ`, `RETRY` |

### 6.3 `integration_pipeline_trigger`

| Column | Type | Notes |
|---|---|---|
| `event_type` | VARCHAR(150) NULL | p13 type |
| `scheduler_job_id` | UUID NULL | p17 soft |
| `manual_allowed` | BOOLEAN | |

---

## 7. Outbound delivery

### 7.1 `integration_outbound_message`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `pipeline_id` | UUID | |
| `connection_id` | UUID | |
| `idempotency_key` | VARCHAR(120) | Unique per tenant |
| `correlation_id` | VARCHAR(64) | |
| `status` | VARCHAR(20) | |
| `trigger_event_id` | UUID NULL | |
| `job_id` | UUID NULL | p14 |

### 7.2 `integration_delivery_attempt`

| Column | Type | Notes |
|---|---|---|
| `outbound_message_id` | UUID | |
| `attempt_no` | INT | |
| `started_at` / `finished_at` | TIMESTAMPTZ | |
| `http_status` | INT NULL | |
| `error_class` | VARCHAR(30) NULL | |
| `error_message` | TEXT NULL | Redacted |
| `latency_ms` | INT NULL | |
| `request_media_id` | UUID NULL | |
| `response_media_id` | UUID NULL | |
| `metadata_fields.adapter` | JSONB | Stub / pending kind (`stub`, `PROVIDER_PENDING`). Not a live vendor client. |

### 7.3 `integration_retry_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | |
| `max_attempts` | INT | |
| `initial_delay_ms` | INT | |
| `max_delay_ms` | INT | |
| `multiplier` | NUMERIC | |
| `jitter` | BOOLEAN | |

### 7.4 `integration_circuit_breaker`

| Column | Type | Notes |
|---|---|---|
| `connection_id` | UUID | |
| `state` | VARCHAR(20) | |
| `failure_threshold` | INT | |
| `open_until` | TIMESTAMPTZ NULL | |
| `success_threshold` | INT | Half-open |

---

## 8. Inbound webhooks

### 8.1 `integration_webhook_endpoint`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `endpoint_key` | VARCHAR(80) | |
| `path_suffix` | VARCHAR(120) | `/hooks/{tenant}/{key}` |
| `connection_id` | UUID NULL | |
| `pipeline_id` | UUID | |
| `is_active` | BOOLEAN | |
| `verify_mode` | VARCHAR(30) | `HMAC`, `JWT`, `BASIC`, `MTLS` |

### 8.2 `integration_inbound_reception`

| Column | Type | Notes |
|---|---|---|
| `endpoint_id` | UUID | |
| `received_at` | TIMESTAMPTZ | |
| `source_ip` | INET NULL | |
| `idempotency_key` | VARCHAR(120) NULL | |
| `status` | VARCHAR(20) | |
| `http_ack_status` | INT | |

Unique `(endpoint_id, idempotency_key)` when key present.

---

## 9. DLQ & reconcile

### 9.1 `integration_dlq_item`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `source_kind` | VARCHAR(20) | `OUTBOUND`, `INBOUND` |
| `source_id` | UUID | |
| `error_class` | VARCHAR(30) | |
| `payload_media_id` | UUID NULL | |
| `enqueued_at` | TIMESTAMPTZ | |
| `status` | VARCHAR(20) | `OPEN`, `REPLAYED`, `DISCARDED` |

### 9.2 `integration_reconcile_object`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `connection_id` | UUID | |
| `canonical_type` | VARCHAR(80) | |
| `canonical_id` | UUID | |
| `external_id` | VARCHAR(200) | |
| `last_sync_at` | TIMESTAMPTZ | |
| `sync_hash` | VARCHAR(64) NULL | |

**Unique:** `(tenant_id, connection_id, canonical_type, canonical_id)` and external unique where required.

### 9.3 `integration_sync_state`

Cursors/watermarks for pull-based connectors (`cursor`, `watermark`, `lag_seconds`).

---

## 10. Partner profiles & packs

### 10.1 `integration_partner_profile`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `partner_id` | UUID | Soft p04 |
| `profile_key` | VARCHAR(80) | |
| `sla_policy_id` | UUID NULL | |
| `is_active` | BOOLEAN | |

### 10.2 Packs

Seed: `sample.rest.outbound.v1`, `sample.webhook.inbound.v1`, placeholders for `gst.einvoice`, `payments.gateway`, `fleet.telematics`.

---

## 11. Plumbing

| Table | Purpose |
|---|---|
| `integration_outbox` | Domain events |
| `integration_idempotency_key` | Admin + delivery keys registry aid |

---

## 12. RLS summary

| Class | Policy |
|---|---|
| Connector catalog | Read auth; manage admin |
| Connections / pipelines / mappings (tenant) | FORCE `tenant_id` |
| Deliveries / receptions / DLQ | FORCE `tenant_id` |
| Partner profiles | FORCE `tenant_id` |

---

## 13. Seed minimum

1. Connectors `REST_JSON`, `WEBHOOK_IN`  
2. Default retry policy (5 attempts, exp backoff)  
3. Default breaker (5 fails → open 60s)  
4. Sample canonical schemas `InvoicePosted`, `PaymentCaptured`  
5. Permissions `integration.*`  
6. Pack `core.samples.v1`  

---

## 14. ER overview

```text
connector ── connections ── auth/tls/secret_refs
                │
pipeline ── versions ── steps/triggers/bindings ── mapping versions
                │
outbound_message ── attempts ── retry/CB
webhook_endpoint ── receptions ── inbound process
dlq ← failed outbound/inbound
partner_profile ── pipelines/connections
reconcile_object / sync_state
packages
```

---

## 15. Implementation notes

1. Resolve secrets at call-time via provider; cache short TTL in memory only.  
2. Truncate error_message; store full response in media when debugging enabled.  
3. Breaker state may also live in Redis; PG is source of truth for audit.  
4. Split models: `catalog`, `connection`, `mapping`, `pipeline`, `outbound`, `inbound`, `dlq`, `partner`, `governance`, `plumbing`.
