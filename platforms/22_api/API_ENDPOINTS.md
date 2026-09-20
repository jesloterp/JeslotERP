# JeslotERP API Platform — Complete API Endpoints

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — key HTTP persists hashed secrets on Postgres; empty list is `[]`. Bulk + Composite on p22. Not Production.  
**Package:** `platforms.p22_api`  
**Base path:** `/api/v1/api`  
**Companion:** [`API_GUIDE.md`](API_GUIDE.md) · [`API_SCHEMA.md`](API_SCHEMA.md)

> This document is the **control-plane API** for catalog, products, keys, limits, and analytics.  
> Domain business APIs live under their platforms; they are **registered** here, not redefined here.

---

## 0. Conventions

### Headers

| Header | Required | Notes |
|---|---|---|
| `Authorization` | Yes (except public portal) | Bearer JWT |
| `X-Tenant-Id` | Yes (tenant APIs) | Subscription/key scope |
| `X-Correlation-Id` | Recommended | Trace |
| `Idempotency-Key` | Mutating admin | Required where policy says |
| `X-Api-Key` | Partner traffic | Alternative to JWT for subscribed clients |

### Envelope

All responses use `StandardResponse`:

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": { "request_id": "…", "correlation_id": "…" }
}
```

### Gateway response headers (enforced on product traffic)

| Header | When |
|---|---|
| `X-RateLimit-Limit` | Always when limited |
| `X-RateLimit-Remaining` | Always when limited |
| `X-RateLimit-Reset` | Unix/reset window |
| `Retry-After` | On `429` |
| `Deprecation` | Deprecated ops/versions |
| `Sunset` | HTTP-date when retiring |

### Common errors

| HTTP | Code | Meaning |
|---|---|---|
| 401 | `AUTH_REQUIRED` | Missing/invalid credential |
| 403 | `FORBIDDEN` | Permission / plan deny |
| 404 | `NOT_FOUND` | Unknown id |
| 409 | `CONFLICT` | Duplicate operation_id / key name |
| 422 | `VALIDATION_ERROR` | Bad payload |
| 429 | `RATE_LIMITED` | Quota/RPS exceeded |
| 503 | `SNAPSHOT_UNAVAILABLE` | Gateway snapshot missing (fail-closed) |

---

## 1. Catalog — services & operations

### 1.1 List services

`GET /api/v1/api/services`

**Permission:** `api.catalog.read`

**Query:** `q`, `is_public`, `owner_platform`, `page`, `page_size`

**Response `data.items[]`:** `id`, `service_key`, `name`, `base_path`, `owner_platform`, `is_public`, `operation_count`

---

### 1.2 Get / create / update service

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/services/{service_id}` | `api.catalog.read` |
| `POST` | `/api/v1/api/services` | `api.catalog.manage` |
| `PATCH` | `/api/v1/api/services/{service_id}` | `api.catalog.manage` |

**POST body:**

```json
{
  "service_key": "document",
  "name": "Document Platform",
  "base_path": "/api/v1/documents",
  "owner_platform": "p09_document",
  "is_public": false
}
```

---

### 1.3 List / register operations

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/services/{service_id}/operations` | `api.catalog.read` |
| `POST` | `/api/v1/api/services/{service_id}/operations` | `api.catalog.manage` |
| `GET` | `/api/v1/api/operations/{operation_id}` | `api.catalog.read` |
| `PATCH` | `/api/v1/api/operations/{operation_id}` | `api.catalog.manage` |
| `POST` | `/api/v1/api/operations/{operation_id}/deprecate` | `api.catalog.manage` |

**POST operation body:**

```json
{
  "operation_id": "createDocument",
  "method": "POST",
  "path_template": "/api/v1/documents",
  "summary": "Create document",
  "permission_code": "document.create",
  "feature_flag_key": null,
  "idempotency_required": true,
  "auth_mode_override": null
}
```

**Bulk register (CI):**

`POST /api/v1/api/catalog/register-batch`  
**Permission:** `api.admin`  
**Body:** `{ "services": [ { "service_key": "…", "operations": [ … ] } ] }`  
Validates uniqueness; upserts; updates CI gate status.

---

## 2. OpenAPI specs & API versions

### 2.1 Specs

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/specs` | `api.catalog.read` |
| `POST` | `/api/v1/api/specs` | `api.catalog.manage` |
| `POST` | `/api/v1/api/specs/{spec_id}/versions` | `api.catalog.manage` |
| `GET` | `/api/v1/api/specs/{spec_id}/versions/{version_id}` | `api.catalog.read` |
| `POST` | `/api/v1/api/specs/{spec_id}/versions/{version_id}/publish` | `api.catalog.manage` |

**Publish version body:** OpenAPI JSON/YAML + `version_label`. Response includes `checksum`.

### 2.2 Product API versions (`v1` / `v2`)

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/versions` | `api.catalog.read` |
| `POST` | `/api/v1/api/versions` | `api.catalog.manage` |
| `POST` | `/api/v1/api/versions/{version_key}/activate` | `api.catalog.manage` |
| `POST` | `/api/v1/api/versions/{version_key}/deprecate` | `api.catalog.manage` |
| `GET` | `/api/v1/api/versions/{version_key}/changelog` | `api.catalog.read` |
| `POST` | `/api/v1/api/versions/{version_key}/changelog` | `api.catalog.manage` |

**Deprecate body:**

```json
{
  "deprecated_at": "2026-10-01T00:00:00Z",
  "sunset_at": "2027-04-01T00:00:00Z",
  "replacement": "v2",
  "migration_notes": "Use /api/v2/… ; field X renamed to Y"
}
```

---

## 3. Products, plans, subscriptions

### 3.1 Products

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/products` | `api.catalog.read` |
| `POST` | `/api/v1/api/products` | `api.product.manage` |
| `PATCH` | `/api/v1/api/products/{product_id}` | `api.product.manage` |
| `PUT` | `/api/v1/api/products/{product_id}/operations` | `api.product.manage` |

**PUT operations:** `{ "operation_ids": ["…", "…"] }` — replaces product operation set.

### 3.2 Plans

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/products/{product_id}/plans` | `api.catalog.read` |
| `POST` | `/api/v1/api/products/{product_id}/plans` | `api.product.manage` |
| `PATCH` | `/api/v1/api/plans/{plan_id}` | `api.product.manage` |
| `PUT` | `/api/v1/api/plans/{plan_id}/limits` | `api.limit.manage` |

**Limits body:**

```json
{
  "limits": [
    { "dimension": "RPS", "limit_value": 50, "window_seconds": 1, "burst": 100 },
    { "dimension": "RPD", "limit_value": 100000, "window_seconds": 86400 }
  ]
}
```

### 3.3 Subscriptions

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/subscriptions` | `api.subscription.manage` |
| `POST` | `/api/v1/api/subscriptions` | `api.subscription.manage` |
| `GET` | `/api/v1/api/subscriptions/{subscription_id}` | `api.subscription.manage` |
| `POST` | `/api/v1/api/subscriptions/{subscription_id}/suspend` | `api.subscription.manage` |
| `POST` | `/api/v1/api/subscriptions/{subscription_id}/resume` | `api.subscription.manage` |
| `POST` | `/api/v1/api/subscriptions/{subscription_id}/change-plan` | `api.subscription.manage` |

**POST create:**

```json
{
  "tenant_id": "…",
  "product_id": "…",
  "plan_id": "…",
  "consumer_id": null,
  "starts_at": "2026-09-09T00:00:00Z"
}
```

RLS: callers only see their tenant unless `api.admin`.

---

## 4. API keys & OAuth bindings

### 4.1 Keys

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/subscriptions/{subscription_id}/keys` | `api.key.manage` |
| `POST` | `/api/v1/api/subscriptions/{subscription_id}/keys` | `api.key.manage` |
| `POST` | `/api/v1/api/keys/{key_id}/revoke` | `api.key.manage` |
| `POST` | `/api/v1/api/keys/{key_id}/rotate` | `api.key.manage` |

**POST create response (one-time):**

```json
{
  "id": "…",
  "name": "Partner prod",
  "key_prefix": "kb_live_ab12",
  "api_key": "kb_live_ab12…FULL_SECRET_ONCE",
  "status": "ACTIVE",
  "expires_at": null
}
```

Subsequent list returns **prefix only**, never full secret.

### 4.2 OAuth client binding

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/api/subscriptions/{subscription_id}/oauth-bindings` | `api.key.manage` |
| `DELETE` | `/api/v1/api/oauth-bindings/{binding_id}` | `api.key.manage` |

**Body:** `{ "oauth_client_id": "<p01 client uuid>", "scopes": ["…"] }`  
Secrets remain in identity; p22 stores binding + product entitlements.

### 4.3 Verify (internal / gateway)

`POST /api/v1/api/internal/keys/verify`  
**Auth:** service / internal  
**Body:** `{ "api_key": "…" }`  
**Response:** subscription, tenant, plan, allowed operation set fingerprint, status.

---

## 5. Rate limits & policies

### 5.1 Policies

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/rate-policies` | `api.limit.manage` |
| `POST` | `/api/v1/api/rate-policies` | `api.limit.manage` |
| `PUT` | `/api/v1/api/limit-bindings` | `api.limit.manage` |
| `GET` | `/api/v1/api/subscriptions/{id}/effective-limits` | `api.limit.manage` |

### 5.2 Check (gateway)

`POST /api/v1/api/internal/rate-limit/check`  
**Body:**

```json
{
  "subscription_id": "…",
  "operation_id": "createDocument",
  "cost": 1
}
```

**Response:**

```json
{
  "allowed": true,
  "limit": 50,
  "remaining": 41,
  "reset_at": "2026-09-09T04:50:01Z"
}
```

On deny: HTTP 200 with `allowed: false` (gateway maps to client `429`) **or** HTTP 429 depending on adapter mode.

### 5.3 Policy bundles & CORS / IP / idempotency

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/policy-bundles` | `api.catalog.manage` |
| `POST` | `/api/v1/api/policy-bundles/{id}/compile` | `api.admin` |
| `PUT` | `/api/v1/api/cors-policies/{id}` | `api.catalog.manage` |
| `PUT` | `/api/v1/api/subscriptions/{id}/ip-allowlist` | `api.subscription.manage` |
| `GET` | `/api/v1/api/idempotency-policies` | `api.catalog.read` |

**IP allowlist body:** `{ "cidrs": ["203.0.113.0/24", "198.51.100.10/32"] }`

---

## 6. Gateway snapshot

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/api/snapshots/publish` | `api.admin` |
| `GET` | `/api/v1/api/snapshots/latest` | `api.admin` |
| `GET` | `/api/v1/api/internal/snapshots/{snapshot_key}` | internal |

**Publish:** compiles catalog + products + policies → `api_gateway_snapshot` + cache tag invalidate.

**Latest response:** `version`, `checksum`, `published_at`, optional `payload` (admin only).

---

## 7. Analytics

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/analytics/usage` | `api.analytics.read` |
| `GET` | `/api/v1/api/analytics/top-routes` | `api.analytics.read` |
| `GET` | `/api/v1/api/analytics/errors` | `api.analytics.read` |
| `GET` | `/api/v1/api/analytics/latency` | `api.analytics.read` |

**Query:** `from`, `to`, `subscription_id`, `operation_id`, `granularity=hour|day`

---

## 8. Developer portal meta

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/portal/pages` | public or `api.portal.manage` |
| `POST` | `/api/v1/api/portal/pages` | `api.portal.manage` |
| `GET` | `/api/v1/api/portal/pages/{slug}` | public if `is_public` |
| `GET` | `/api/v1/api/portal/sdk-links` | public |
| `PUT` | `/api/v1/api/portal/tryit/{operation_id}` | `api.portal.manage` |

Public portal never returns private operation defs or tenant keys.

---

## 9. Governance, packs, CI

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/api/packages` | `api.admin` |
| `POST` | `/api/v1/api/packages/{package_key}/apply` | `api.admin` |
| `GET` | `/api/v1/api/ci/gates` | `api.admin` |
| `POST` | `/api/v1/api/ci/gates/validate` | `api.admin` |
| `GET` | `/api/v1/api/changesets` | `api.catalog.manage` |
| `POST` | `/api/v1/api/changesets/{id}/approve` | `api.admin` |

**CI validate body:** declared operation list from platform OpenAPI extract; response lists missing/extra vs catalog.

---

## 10. Runtime partner traffic (illustrative — not under `/api/v1/api`)

Partners call **domain** routes, e.g.:

`POST /api/v1/documents`  
Headers: `X-Api-Key` + optional `Idempotency-Key`

**Gateway pipeline (p22-enforced):**

1. Resolve key → subscription  
2. Check plan includes operation  
3. Feature flag (p12) if bound  
4. AuthZ permission if JWT path  
5. Rate limit / quota  
6. IP allowlist / CORS  
7. Forward to platform handler  
8. Meter usage (async)

---

## 11. Permission matrix (summary)

| Surface | Min permission |
|---|---|
| Catalog read | `api.catalog.read` |
| Catalog write | `api.catalog.manage` |
| Products/plans | `api.product.manage` |
| Subscriptions | `api.subscription.manage` |
| Keys | `api.key.manage` |
| Limits | `api.limit.manage` |
| Analytics | `api.analytics.read` |
| Portal | `api.portal.manage` |
| Snapshots/packs/CI | `api.admin` |
| Bulk jobs / Composite | `api.catalog.manage` |
| Internal verify/check | service principal |

---

## 12. Example flows

### 12.1 Onboard partner

1. `POST /products` + attach operations  
2. `POST /products/{id}/plans` + limits  
3. `POST /subscriptions` for tenant  
4. `POST /subscriptions/{id}/keys` → store secret once  
5. `POST /snapshots/publish`

### 12.2 Deprecate v1 operation

1. `POST /operations/{id}/deprecate` with sunset  
2. Update OpenAPI version  
3. Publish snapshot  
4. Clients receive `Deprecation` / `Sunset` headers

### 12.3 Rotate key

1. `POST /keys/{id}/rotate` → new secret once; old enters `ROTATING` grace  
2. Partner switches  
3. `POST /keys/{old}/revoke`

---

## 13. Bulk + Composite (HYG-017)

Stay on **p22**. OData `$metadata` / `$batch` and GraphQL are **not** shipped unless a customer demands them.

### 13.1 Composite (sync)

`POST /api/v1/api/composite`  
**Permission:** `api.catalog.manage`

```json
{
  "allOrNone": false,
  "compositeRequest": [
    { "method": "POST", "url": "/services", "referenceId": "svc", "body": { "service_key": "docs", "name": "Docs" } },
    { "method": "GET", "url": "/services/@{svc.id}", "referenceId": "got" }
  ]
}
```

- Max 25 subrequests. Empty `compositeRequest` is `422`.  
- Allow-listed catalog paths only (`/health`, `/services`, `/services/{id}`, `/products`).  
- `@{referenceId.field}` interpolates a prior subrequest body.  
- `allOrNone=true` stops after the first non-2xx; later items are skipped (no store rollback).  
- Response `data.compositeResponse[]`: `referenceId`, `httpStatusCode`, `body`.

### 13.2 Bulk jobs

`POST /api/v1/api/bulk/jobs` creates an `OPEN` job. Empty `GET /api/v1/api/bulk/jobs` is `[]`.

| Method | Path | Notes |
|---|---|---|
| `GET` | `/api/v1/api/bulk/jobs` | Empty catalog is `[]` |
| `POST` | `/api/v1/api/bulk/jobs` | `{ "object": "Service"\|"Product", "operation": "insert", "records": [] }` |
| `GET` | `/api/v1/api/bulk/jobs/{id}` | |
| `POST` | `/api/v1/api/bulk/jobs/{id}/batches` | Queue records while `OPEN` |
| `POST` | `/api/v1/api/bulk/jobs/{id}/close` | Process inserts; per-row success/fail |
| `POST` | `/api/v1/api/bulk/jobs/{id}/abort` | Only while `OPEN` |

**Permission:** `api.catalog.manage`  
Unknown object/operation → `422`. Close/abort of a non-`OPEN` job → `409`. Jobs are process-local (no new `api_*` table).

---

## 14. Related documents

- Guide: [`API_GUIDE.md`](API_GUIDE.md)  
- Schema: [`API_SCHEMA.md`](API_SCHEMA.md)  
- Identity: [`../01_identity/IDENTITY_API.md`](../01_identity/IDENTITY_API.md)  
- Feature: [`../12_feature/FEATURE_API.md`](../12_feature/FEATURE_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
