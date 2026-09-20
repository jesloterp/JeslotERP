# JeslotERP API Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `api_key.key_hash` is the HTTP credential ledger. Counters stay on RateLimiter. Not Production.  
**Package:** `platforms.p22_api`  
**PostgreSQL schema:** `api`  
**Companion:** [`API_GUIDE.md`](API_GUIDE.md) · [`API_ENDPOINTS.md`](API_ENDPOINTS.md)

> Runtime models: `platforms/p22_api/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `api` (never `p22`) |
| Tables | `api_*` |
| Soft delete | Revoke keys; retire versions |
| Cross-schema | UUID refs (tenant, oauth client) |
| RLS | FORCE on tenant subscriptions/keys/usage |
| Secrets | API key **hash** only; OAuth secrets in identity/config |

---

## 2. Complete table inventory (**60 tables**)

### 2.1 Catalog (10)

| # | Table | Purpose |
|---|---|---|
| 1 | `api_service` | API services |
| 2 | `api_operation` | Operations (method+path) |
| 3 | `api_operation_tag` | Tags |
| 4 | `api_path_param` | Path params |
| 5 | `api_query_param` | Query params |
| 6 | `api_header_param` | Header requirements |
| 7 | `api_request_body` | Body content types |
| 8 | `api_response` | Response codes |
| 9 | `api_error_code` | Standard error codes |
| 10 | `api_feature_binding` | Feature gates per op |

### 2.2 Specs & versions (7)

| # | Table | Purpose |
|---|---|---|
| 11 | `api_spec` | OpenAPI documents |
| 12 | `api_spec_version` | Spec versions |
| 13 | `api_api_version` | v1/v2 product versions |
| 14 | `api_revision` | Non-breaking revisions |
| 15 | `api_changelog` | Changelog entries |
| 16 | `api_deprecation` | Deprecation records |
| 17 | `api_sunset_policy` | Sunset rules |

### 2.3 Products, plans, subscriptions (8)

| # | Table | Purpose |
|---|---|---|
| 18 | `api_product` | Products |
| 19 | `api_product_operation` | Product ↔ operations |
| 20 | `api_plan` | Plans |
| 21 | `api_plan_limit` | Limits on plan |
| 22 | `api_subscription` | Tenant/consumer subscriptions |
| 23 | `api_subscription_operation_override` | Allow/deny overrides |
| 24 | `api_consumer` | Partner/consumer orgs |
| 25 | `api_entitlement_hook` | Optional licensing bridge |

### 2.4 Credentials (7)

| # | Table | Purpose |
|---|---|---|
| 26 | `api_key` | API keys (hashed) |
| 27 | `api_key_scope` | Scopes/ops grants |
| 28 | `api_oauth_client_binding` | Link to identity OAuth client |
| 29 | `api_credential_audit` | Issue/revoke audit meta |
| 30 | `api_key_rotation` | Rotation jobs |
| 31 | `api_mtls_binding` | Optional mTLS client cert refs |
| 32 | `api_ingest_principal` | Machine principals |

### 2.5 Rate limits & quotas (7)

| # | Table | Purpose |
|---|---|---|
| 33 | `api_rate_policy` | Rate policies |
| 34 | `api_quota_policy` | Quota policies |
| 35 | `api_limit_binding` | Bind to plan/key/op |
| 36 | `api_limit_counter_meta` | Counter key templates |
| 37 | `api_burst_policy` | Burst tokens |
| 38 | `api_concurrency_policy` | Max in-flight |
| 39 | `api_throttle_event` | Deny samples |

### 2.6 Gateway policies (8)

| # | Table | Purpose |
|---|---|---|
| 40 | `api_policy_bundle` | Compiled policy sets |
| 41 | `api_auth_policy` | JWT/API_KEY/etc |
| 42 | `api_idempotency_policy` | Idempotency requirements |
| 43 | `api_cors_policy` | CORS |
| 44 | `api_ip_allowlist` | IP lists |
| 45 | `api_payload_policy` | Max sizes |
| 46 | `api_header_policy` | Required headers |
| 47 | `api_gateway_snapshot` | Published snapshots |

### 2.7 Analytics, portal, governance (13)

| # | Table | Purpose |
|---|---|---|
| 48 | `api_usage_rollup` | Usage aggregates |
| 49 | `api_usage_top_route` | Top routes |
| 50 | `api_latency_rollup` | Latency |
| 51 | `api_error_rollup` | Errors |
| 52 | `api_portal_page` | Portal pages meta |
| 53 | `api_portal_tryit` | Try-it configs |
| 54 | `api_sdk_link` | SDK download links |
| 55 | `api_changeset` | Changes |
| 56 | `api_approval` | Approvals |
| 57 | `api_package` | Packs |
| 58 | `api_package_item` | Items |
| 59 | `api_catalog_audit` | Audit |
| 60 | `api_ci_gate` | CI registration gates |

**Plumbing:** `api_outbox`, `api_idempotency_key`

**Implementation total with plumbing: 62 tables.**

---

## 3. Enumerations (selected)

| Enum | Values |
|---|---|
| `api_http_method` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS` |
| `api_version_lifecycle` | `DRAFT`, `PUBLISHED`, `ACTIVE`, `DEPRECATED`, `RETIRED` |
| `api_auth_mode` | `NONE`, `JWT`, `API_KEY`, `JWT_OR_API_KEY`, `MTLS` |
| `api_limit_dim` | `RPS`, `RPM`, `RPD`, `CONCURRENCY`, `BANDWIDTH` |
| `api_key_status` | `ACTIVE`, `REVOKED`, `EXPIRED`, `ROTATING` |
| `api_subscription_status` | `ACTIVE`, `SUSPENDED`, `CANCELLED` |
| `api_deprecation_state` | `ANNOUNCED`, `DEPRECATED`, `SUNSET` |

---

## 4. Catalog detail

### 4.1 `api_service`

| Column | Type | Notes |
|---|---|---|
| `service_key` | VARCHAR(80) UNIQUE | `identity`, `document` |
| `name` | VARCHAR(150) | |
| `base_path` | VARCHAR(100) | `/api/v1` |
| `owner_platform` | VARCHAR(80) | |
| `is_public` | BOOLEAN | |

### 4.2 `api_operation`

| Column | Type | Notes |
|---|---|---|
| `service_id` | UUID | |
| `operation_id` | VARCHAR(120) | Unique per service |
| `method` | VARCHAR(10) | |
| `path_template` | VARCHAR(300) | |
| `summary` | VARCHAR(200) NULL | |
| `is_deprecated` | BOOLEAN | |
| `auth_mode_override` | VARCHAR(30) NULL | |
| `permission_code` | VARCHAR(100) NULL | IAM permission hint |
| `feature_flag_key` | VARCHAR(150) NULL | |
| `idempotency_required` | BOOLEAN | |

**Unique:** `(service_id, operation_id)` and `(service_id, method, path_template)`.

---

## 5. Specs & versions

### 5.1 `api_spec_version`

| Column | Type | Notes |
|---|---|---|
| `spec_id` | UUID | |
| `version_label` | VARCHAR(50) | `2026.09.09` |
| `openapi_version` | VARCHAR(10) | `3.1.0` |
| `content` | JSONB/TEXT | Spec doc |
| `checksum` | VARCHAR(64) | |
| `lifecycle` | VARCHAR(20) | |

### 5.2 `api_api_version`

| Column | Type | Notes |
|---|---|---|
| `version_key` | VARCHAR(20) UNIQUE | `v1`, `v2` |
| `lifecycle` | VARCHAR(20) | |
| `base_path` | VARCHAR(50) | `/api/v1` |
| `sunset_at` | TIMESTAMPTZ NULL | |

### 5.3 `api_deprecation`

| Column | Type | Notes |
|---|---|---|
| `version_id` / `operation_id` | | Target |
| `announced_at` | TIMESTAMPTZ | |
| `deprecated_at` | TIMESTAMPTZ | |
| `sunset_at` | TIMESTAMPTZ | |
| `replacement` | VARCHAR(200) NULL | |
| `migration_notes` | TEXT NULL | |

---

## 6. Products & subscriptions

### 6.1 `api_product`

| Column | Type | Notes |
|---|---|---|
| `product_key` | VARCHAR(80) UNIQUE | `partner_erp` |
| `name` | VARCHAR(150) | |
| `description` | TEXT NULL | |
| `is_active` | BOOLEAN | |

### 6.2 `api_plan`

| Column | Type | Notes |
|---|---|---|
| `product_id` | UUID | |
| `plan_key` | VARCHAR(50) | `standard` |
| `name` | VARCHAR(100) | |
| `is_default` | BOOLEAN | |

### 6.3 `api_subscription`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NOT NULL | RLS |
| `consumer_id` | UUID NULL | |
| `product_id` | UUID | |
| `plan_id` | UUID | |
| `status` | VARCHAR(20) | |
| `starts_at` / `ends_at` | TIMESTAMPTZ | |

---

## 7. API keys

### 7.1 `api_key`

| Column | Type | Notes |
|---|---|---|
| `subscription_id` | UUID | |
| `tenant_id` | UUID | |
| `name` | VARCHAR(100) | |
| `key_prefix` | VARCHAR(12) | `kb_live_ab12` display |
| `key_hash` | VARCHAR(64) | SHA-256 |
| `status` | VARCHAR(20) | |
| `expires_at` | TIMESTAMPTZ NULL | |
| `last_used_at` | TIMESTAMPTZ NULL | |
| `created_by` | UUID NULL | |

Never store plaintext. Prefix for UI identification only.

---

## 8. Limits

### 8.1 `api_rate_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(50) | |
| `dimension` | VARCHAR(20) | RPS/RPM/… |
| `limit_value` | INT | |
| `window_seconds` | INT | |
| `burst` | INT NULL | |

### 8.2 `api_limit_binding`

Binds policy to `plan_id` / `subscription_id` / `operation_id` / `key_id` with priority.

Counter keys (Redis): `rl:{subscription}:{op}:{window}`.

---

## 9. Policies & snapshots

### 9.1 `api_policy_bundle`

| Column | Type | Notes |
|---|---|---|
| `bundle_key` | VARCHAR(80) | |
| `version` | INT | |
| `checksum` | VARCHAR(64) | |
| `compiled` | JSONB | Gateway-ready |

### 9.2 `api_gateway_snapshot`

| Column | Type | Notes |
|---|---|---|
| `snapshot_key` | VARCHAR(80) | `production` |
| `version` | INT | |
| `checksum` | VARCHAR(64) | |
| `payload` | JSONB | Full compiled catalog+policies |
| `published_at` | TIMESTAMPTZ | |

Gateway/cache loads latest snapshot; invalidate on publish.

---

## 10. Analytics & portal

### 10.1 `api_usage_rollup`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID | |
| `subscription_id` | UUID NULL | |
| `operation_id` | UUID NULL | |
| `window_start` | TIMESTAMPTZ | |
| `requests` | BIGINT | |
| `errors_4xx` / `errors_5xx` | BIGINT | |
| `bytes_in` / `bytes_out` | BIGINT | |

### 10.2 `api_portal_page`

| Column | Type | Notes |
|---|---|---|
| `slug` | VARCHAR(100) | |
| `title` | VARCHAR(150) | |
| `spec_version_id` | UUID NULL | |
| `is_public` | BOOLEAN | |

---

## 11. Governance & packs

- Packs seed `core.public.v1` product with identity login + org read ops  
- Internal product for service-to-service  
- CI gate table lists required registered operation_ids per platform  

---

## 12. Plumbing

| Table | Purpose |
|---|---|
| `api_outbox` | Domain events |
| `api_idempotency_key` | Admin APIs |

---

## 13. RLS summary

| Class | Policy |
|---|---|
| Catalog/specs/products | Read auth; manage permission |
| Subscriptions/keys/usage | FORCE `tenant_id` |
| Snapshots | Admin publish; gateways read via internal |

---

## 14. Seed minimum

1. Services for live platforms (identity, org, configuration, …)  
2. `v1` ACTIVE version  
3. Products `internal`, `tenant_admin`  
4. Plans with baseline RPS  
5. Idempotency policy for POST mutating ops  
6. CORS default for admin UI origin placeholder  
7. Permissions `api.*`  
8. Error code catalog aligning StandardResponse  

---

## 15. ER overview

```text
service ── operations ── params/responses
spec ── versions
api_version ── deprecations / changelogs
product ── operations
plan ── limits
subscription ── keys / oauth bindings
policy_bundle ── gateway_snapshot
usage_rollups / portal
packages
```

---

## 16. Implementation notes

1. Verify API key: hash presented secret with SHA-256 (+ pepper from config).  
2. Snapshot compile must be deterministic for checksum.  
3. Rate limiter fail-open vs fail-closed is env policy (prod fail-closed for public).  
4. Split models: `catalog`, `spec`, `product`, `credential`, `limit`, `policy`, `analytics`, `governance`, `plumbing`.  
5. HYG-017 Bulk jobs are process-local on `ApiCatalogStore.bulk_jobs` (no new encyclopedia table). Composite does not persist.
