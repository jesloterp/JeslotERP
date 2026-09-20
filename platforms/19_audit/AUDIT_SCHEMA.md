# JeslotERP Audit Platform — Production Schema (Advanced)

**Version:** 1.2 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `aud_event` HTTP persist is append-only. SIEM delivery is a port; live HTTP is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p19_audit`  
**PostgreSQL schema:** `audit`  
**Companion:** [`AUDIT_GUIDE.md`](AUDIT_GUIDE.md) · [`AUDIT_API.md`](AUDIT_API.md)

> Runtime models: `platforms/p19_audit/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `audit` (never `p19`) |
| Tables | `aud_*` |
| Mutability | Event tables **INSERT-only** |
| Soft delete | N/A for events; catalog may retire |
| Cross-schema | UUID refs only |
| RLS | FORCE on tenant events for readers |
| Hashes | SHA-256 hex |

---

## 2. Complete table inventory (**60 tables**)

### 2.1 Catalogs (9)

| # | Table | Purpose |
|---|---|---|
| 1 | `aud_action` | Action catalog |
| 2 | `aud_action_category` | AUTH/DATA/ADMIN/SECURITY/… |
| 3 | `aud_object_type` | Object/entity types |
| 4 | `aud_outcome` | SUCCESS/FAIL/DENIED |
| 5 | `aud_actor_type` | USER/SYSTEM/SERVICE/… |
| 6 | `aud_sensitivity` | Event sensitivity classes |
| 7 | `aud_field_policy` | Which fields to capture |
| 8 | `aud_platform` | Emitting platforms |
| 9 | `aud_feature_binding` | Feature gates |

### 2.2 Events & diffs (8)

| # | Table | Purpose |
|---|---|---|
| 10 | `aud_event` | Immutable event header |
| 11 | `aud_event_data` | Summary JSON (non-PII preferred) |
| 12 | `aud_field_change` | Field diffs |
| 13 | `aud_event_tag` | Tags |
| 14 | `aud_event_link` | Links to related objects |
| 15 | `aud_event_geo` | Optional geo/ip enrich |
| 16 | `aud_event_error` | Failure details |
| 17 | `aud_event_attachment` | Optional evidence media_id |

### 2.3 Integrity (6)

| # | Table | Purpose |
|---|---|---|
| 18 | `aud_stream` | Hash streams |
| 19 | `aud_chain_pointer` | Head hash per stream |
| 20 | `aud_seal` | Batch seals |
| 21 | `aud_seal_item` | Events in seal |
| 22 | `aud_verify_run` | Verification jobs |
| 23 | `aud_tamper_alert` | Tamper detections |

### 2.4 Ingest (7)

| # | Table | Purpose |
|---|---|---|
| 24 | `aud_ingest_source` | Sources |
| 25 | `aud_event_binding` | p13 type → action mapping |
| 26 | `aud_ingest_batch` | Batches |
| 27 | `aud_ingest_dead` | Poison ingest |
| 28 | `aud_writer_grant` | Who may write |
| 29 | `aud_idempotency` | Source idempotency |
| 30 | `aud_enrichment` | Enrichment handlers |

### 2.5 Redaction & views (5)

| # | Table | Purpose |
|---|---|---|
| 31 | `aud_redaction_policy` | Policies |
| 32 | `aud_redaction_rule` | Field/regex rules |
| 33 | `aud_mask_profile` | Mask styles |
| 34 | `aud_break_glass` | Break-glass sessions |
| 35 | `aud_view_preference` | Investigator prefs |

### 2.6 Retention & legal hold (7)

| # | Table | Purpose |
|---|---|---|
| 36 | `aud_retention_policy` | Retention |
| 37 | `aud_retention_binding` | Bind to action/object |
| 38 | `aud_legal_hold` | Holds |
| 39 | `aud_legal_hold_filter` | Scope filters |
| 40 | `aud_purge_job` | Purge runs |
| 41 | `aud_purge_item` | Items purged (meta only) |
| 42 | `aud_archive_tier` | Cold archive refs |

### 2.7 Query, export, alerts (10)

| # | Table | Purpose |
|---|---|---|
| 43 | `aud_saved_query` | Saved investigations |
| 44 | `aud_export_case` | Export cases |
| 45 | `aud_export_item` | Included events |
| 46 | `aud_export_artifact` | media_id + checksum |
| 47 | `aud_alert_rule` | Alert rules |
| 48 | `aud_alert_event` | Fired alerts |
| 49 | `aud_siem_endpoint` | SIEM/webhook endpoints |
| 50 | `aud_siem_delivery` | Delivery ledger |
| 51 | `aud_access_event` | Audit-of-audit |
| 52 | `aud_query_stats` | Query metrics |

### 2.8 Governance & packs (8)

| # | Table | Purpose |
|---|---|---|
| 53 | `aud_changeset` | Catalog changes |
| 54 | `aud_approval` | Approvals |
| 55 | `aud_package` | Packs |
| 56 | `aud_package_item` | Items |
| 57 | `aud_simulation_run` | Hash/redaction sims |
| 58 | `aud_catalog_audit` | Catalog audit |
| 59 | `aud_partition_policy` | Time partitioning |
| 60 | `aud_storage_backend` | Optional cold store |

**Plumbing:** `aud_outbox`, `aud_idempotency_key` (API)

**Implementation total with plumbing: 62 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `aud_outcome_code` | `SUCCESS`, `FAIL`, `DENIED`, `PARTIAL` |
| `aud_actor_code` | `USER`, `SYSTEM`, `SERVICE`, `SUPPORT`, `BREAK_GLASS` |
| `aud_sensitivity_code` | `NORMAL`, `SENSITIVE`, `RESTRICTED` |
| `aud_ingest_status` | `ACCEPTED`, `DUPLICATE`, `REJECTED`, `DEAD` |
| `aud_hold_status` | `ACTIVE`, `RELEASED` |
| `aud_purge_status` | `PENDING`, `RUNNING`, `COMPLETED`, `BLOCKED`, `FAILED` |
| `aud_export_status` | `OPEN`, `PACKAGING`, `READY`, `FAILED`, `EXPIRED` |
| `aud_seal_status` | `OPEN`, `SEALED`, `VERIFIED`, `INVALID` |

---

## 4. Catalog detail

### 4.1 `aud_action`

| Column | Type | Notes |
|---|---|---|
| `action_key` | VARCHAR(80) UNIQUE | `USER_LOGIN`, `ENTITY_UPDATE`, `PERMISSION_GRANT` |
| `category_id` | UUID | |
| `name` | VARCHAR(150) | |
| `sensitivity` | VARCHAR(20) | |
| `default_retain_years` | INT NULL | |
| `capture_fields` | BOOLEAN | |
| `label_key` | VARCHAR(200) NULL | |

### 4.2 `aud_object_type`

| Column | Type | Notes |
|---|---|---|
| `object_type_key` | VARCHAR(100) UNIQUE | `sales.order`, `identity.user` |
| `metadata_entity_key` | VARCHAR(100) NULL | |
| `name` | VARCHAR(150) | |

### 4.3 `aud_field_policy`

| Column | Type | Notes |
|---|---|---|
| `object_type_id` | UUID | |
| `field_key` | VARCHAR(100) | |
| `capture` | BOOLEAN | |
| `mask_profile_id` | UUID NULL | |
| `never_capture` | BOOLEAN | passwords, secrets |

---

## 5. Events (immutable)

### 5.1 `aud_event`

| Column | Type | Notes |
|---|---|---|
| `event_id` | UUID PK | |
| `occurred_at` | TIMESTAMPTZ NOT NULL | |
| `ingested_at` | TIMESTAMPTZ NOT NULL | |
| `tenant_id` | UUID NOT NULL | RLS |
| `company_id` | UUID NULL | |
| `actor_type` | VARCHAR(20) | |
| `actor_id` | UUID NULL | |
| `actor_display` | VARCHAR(200) NULL | Denorm snapshot |
| `action_key` | VARCHAR(80) | |
| `object_type` | VARCHAR(100) NULL | |
| `object_id` | UUID NULL | |
| `object_display` | VARCHAR(200) NULL | |
| `outcome` | VARCHAR(20) | |
| `request_id` | UUID NULL | |
| `correlation_id` | UUID NULL | |
| `causation_id` | UUID NULL | |
| `session_id` | UUID NULL | |
| `ip` | INET NULL | |
| `user_agent` | TEXT NULL | |
| `source_platform` | VARCHAR(80) | |
| `source_event_id` | VARCHAR(120) NULL | |
| `stream_id` | UUID | |
| `sequence_no` | BIGINT | Per stream |
| `prev_hash` | VARCHAR(64) | |
| `event_hash` | VARCHAR(64) | |
| `sensitivity` | VARCHAR(20) | |

**Unique:** `(source_platform, source_event_id)` where source_event_id not null.  
**Unique:** `(stream_id, sequence_no)`.

**Triggers/grants:** block UPDATE/DELETE.

### 5.2 `aud_field_change`

| Column | Type | Notes |
|---|---|---|
| `event_id` | UUID | |
| `field_key` | VARCHAR(100) | |
| `old_value` | TEXT NULL | Already redacted for storage policy |
| `new_value` | TEXT NULL | |
| `value_type` | VARCHAR(20) | |
| `is_masked` | BOOLEAN | |

### 5.3 `aud_event_data`

Small JSON summary (ids, amounts as numbers if allowed); large blobs forbidden.

---

## 6. Integrity

### 6.1 `aud_stream`

| Column | Type | Notes |
|---|---|---|
| `stream_key` | VARCHAR(100) | e.g. `tenant:{id}` |
| `tenant_id` | UUID NULL | |
| `algorithm` | VARCHAR(20) DEFAULT 'SHA256' | |

### 6.2 `aud_seal`

| Column | Type | Notes |
|---|---|---|
| `stream_id` | UUID | |
| `from_seq` / `to_seq` | BIGINT | |
| `seal_hash` | VARCHAR(64) | |
| `sealed_at` | TIMESTAMPTZ | |
| `status` | VARCHAR(20) | |

---

## 7. Ingest

### 7.1 `aud_event_binding`

| Column | Type | Notes |
|---|---|---|
| `event_type_key` | VARCHAR(200) | p13 |
| `action_key` | VARCHAR(80) | |
| `object_type_path` | VARCHAR(100) NULL | JSONPath-ish to entity |
| `is_active` | BOOLEAN | |

### 7.2 `aud_writer_grant`

| Column | Type | Notes |
|---|---|---|
| `service_key` | VARCHAR(80) | |
| `action_prefix` | VARCHAR(80) | `DOCUMENT_%` |
| `is_active` | BOOLEAN | |

---

## 8. Redaction & break-glass

### 8.1 `aud_redaction_rule`

| Column | Type | Notes |
|---|---|---|
| `policy_id` | UUID | |
| `field_key` | VARCHAR(100) NULL | |
| `pattern` | VARCHAR(200) NULL | e.g. PAN/GSTIN |
| `mask_profile_id` | UUID | |

### 8.2 `aud_break_glass`

| Column | Type | Notes |
|---|---|---|
| `user_id` | UUID | |
| `reason` | TEXT | |
| `ticket_ref` | VARCHAR(100) NULL | |
| `expires_at` | TIMESTAMPTZ | |
| `approved_by` | UUID NULL | |

---

## 9. Retention & hold

### 9.1 `aud_retention_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | `IN_FINANCIAL_7Y` |
| `retain_years` | INT | |
| `archive_after_years` | INT NULL | |
| `action_category` | VARCHAR(30) NULL | |

### 9.2 `aud_legal_hold`

| Column | Type | Notes |
|---|---|---|
| `hold_key` | VARCHAR(100) | |
| `tenant_id` | UUID | |
| `reason` | TEXT | |
| `is_active` | BOOLEAN | |
| `applied_by` | UUID | |

Filters may constrain object_type/action/time range.

### 9.3 `aud_purge_job`

Only deletes/archives events past retention with **no active hold**; writes purge meta without restoring content.

---

## 10. Export & SIEM

### 10.1 `aud_export_case`

| Column | Type | Notes |
|---|---|---|
| `case_key` | VARCHAR(100) | |
| `tenant_id` | UUID | |
| `query_json` | JSONB | |
| `status` | VARCHAR(20) | |
| `artifact_media_id` | UUID NULL | |
| `checksum` | VARCHAR(64) NULL | |
| `created_by` | UUID | |

### 10.2 `aud_siem_endpoint`

| Column | Type | Notes |
|---|---|---|
| `endpoint_key` | VARCHAR(50) | |
| `url` | TEXT | |
| `secret_ref_key` | VARCHAR(150) | Vault ref only — never a plaintext webhook secret |
| `is_active` | BOOLEAN | |

SIEM forward is a `SiemForwarder` port. Pytest/dev uses `StubSiemForwarder` (in-process signed ledger). `HttpxSiemForwarder` never invents DELIVERED — ping/forward are `PROVIDER_PENDING` until a real vendor webhook + credentials exist. Endpoint/delivery HTTP catalog stays memory in this slice; event ingest SoR is `aud_event`.

---

## 11. Access events (audit-of-audit)

### 11.1 `aud_access_event`

| Column | Type | Notes |
|---|---|---|
| `actor_id` | UUID | |
| `access_kind` | VARCHAR(30) | QUERY/EXPORT/BREAK_GLASS/VERIFY |
| `query_hash` | VARCHAR(64) NULL | |
| `result_count` | INT NULL | |
| `sensitive` | BOOLEAN | |
| `occurred_at` | TIMESTAMPTZ | |

Also hash-chained in a dedicated stream optionally.

---

## 12. Governance & packs

- `audit.core.actions@1.0.0` — identity/org/document/process actions  
- `audit.india.retention@1.0.0` — financial retention bindings  
- Partition policy by month on `occurred_at`  

---

## 13. Plumbing

| Table | Purpose |
|---|---|
| `aud_outbox` | Low-volume meta events |
| `aud_idempotency_key` | Export/hold APIs |

---

## 14. RLS summary

| Class | Policy |
|---|---|
| Catalog | Read auth; admin manage |
| Events/diffs | FORCE `tenant_id` |
| Export/hold | FORCE tenant |
| Writer grants | Admin |
| Break-glass | Strict + access event |

---

## 15. Seed minimum

1. Action categories + core actions (login, CRUD, permission, publish, allocate, kill_switch)  
2. Object types for identity/org/document/process/feature  
3. Retention defaults  
4. Redaction masks for email/phone/PAN/GSTIN  
5. Stream-per-tenant policy  
6. Permissions `audit.*`  
7. Writer grants for platform services  

---

## 16. ER overview

```text
action / object_type / field_policy
ingest bindings / writer grants
event ── field_changes / data / tags / links
     ── stream sequence + hashes
seal ── verify_run
retention / legal_hold / purge
export_case / siem / alerts
access_event
packages
```

---

## 17. Implementation notes

1. Partition `aud_event` by month; attach partitions via policy.  
2. Compute `event_hash` over canonical canonicalized payload + prev_hash.  
3. Store pre-masked field values; unmask only via break-glass projection from separate vaulted store if ever needed — default is irreversible mask.  
4. Prefer irreversible redaction for secrets; don’t keep parallel plaintext.  
5. Split models: `catalog`, `event`, `integrity`, `ingest`, `retention`, `export`, `security`, `governance`, `plumbing`.
