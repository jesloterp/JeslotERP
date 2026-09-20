# JeslotERP Licensing Platform — Complete API Endpoints (V2)

**Version:** 2.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — products/subscriptions/entitlements are Postgres-first; empty list is `[]`. Live PSP is `PROVIDER_PENDING`. Not Production.  
**Package:** `platforms.p26_licensing`  
**Base path (normative):** `/api/v1/licensing`  
**Companion:** [`V2_LICENSING_GUIDE.md`](V2_LICENSING_GUIDE.md) · [`V2_LICENSING_SCHEMA.md`](V2_LICENSING_SCHEMA.md)  
**v1 encyclopedia (archived REST):** [`LICENSING_API.md`](LICENSING_API.md) — table of historical paths only. Bare `/api/v1/products*` is **410** `LICENSING_V1_GONE`.

> Register this surface in **p22**. Dedicated licensing host may still use the same path prefix.

---

## 0. Conventions

### Headers

| Header | Required | Notes |
|---|---|---|
| `Authorization` | Yes | Bearer JWT from `p01_identity` |
| `X-Correlation-Id` | Recommended | Trace |
| `Idempotency-Key` | Commerce mutations | Checkout confirm, usage ingest, subscribe |
| `If-Match` | Concurrent updates | Integer `version` |

**Tenant:** derived from verified token/security context. Never treat client `X-Tenant-Id` as authoritative alone.

### Envelope (`StandardResponse`)

```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": { "request_id": "…", "correlation_id": "…", "version": 3 }
}
```

v1 response shapes map into `data` / `error.code` without changing domain fields.

### Common errors

| HTTP | Code | Meaning |
|---|---|---|
| 401 | `AUTH_REQUIRED` | Missing/invalid JWT |
| 403 | `FORBIDDEN` / `NOT_ENTITLED` | RBAC or entitlement deny |
| 404 | `NOT_FOUND` | Unknown id |
| 409 | `CONFLICT` / `VERSION_CONFLICT` | Idempotency or `If-Match` |
| 402 | `PAYMENT_REQUIRED` | Soft-block / past_due — **deferred until billing (Phase G)**. Runtime checks today are entitled/limit only. |
| 422 | `VALIDATION_ERROR` | Bad payload |
| 429 | `QUOTA_EXCEEDED` / `RATE_LIMITED` | `check-limit` remaining < requested; usage meters |
| 423 | `SUBSCRIPTION_BLOCKED` | Hard dunning suspend — **deferred until billing (Phase G)**. |

### Path migration

| Legacy (v1, **410 Gone**) | V2 normative |
|---|---|
| `GET /api/v1/products` | `GET /api/v1/licensing/products` |
| `GET /api/v1/product-versions/{id}` | `GET /api/v1/licensing/product-versions/{id}` |
| `GET /api/v1/product-categories` | catalog under `/api/v1/licensing/*` |
| `POST /api/v1/usage/events` | `POST /api/v1/licensing/usage/events` |
| `…/entitlement/check-feature` | same under `/licensing` prefix |

---

## 1. Catalog — products, features, modules

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/licensing/products` | `licensing.catalog.read` |
| `POST` | `/api/v1/licensing/products` | `licensing.catalog.manage` |
| `GET` | `/api/v1/licensing/products/{id}` | `licensing.catalog.read` |
| `PATCH` | `/api/v1/licensing/products/{id}` | `licensing.catalog.manage` |
| `POST` | `/api/v1/licensing/products/{id}/versions` | `licensing.catalog.manage` |
| `POST` | `/api/v1/licensing/product-versions/{id}/publish` | `licensing.catalog.manage` |
| `GET/POST` | `/api/v1/licensing/features` | read / manage |
| `GET/POST` | `/api/v1/licensing/modules` | read / manage |
| `GET` | `/api/v1/licensing/currencies` | `licensing.catalog.read` |
| `GET/POST` | `/api/v1/licensing/units` | read / manage |

---

## 2. Plans, prices, add-ons

| Method | Path | Permission |
|---|---|---|
| `GET/POST` | `/api/v1/licensing/plans` | catalog |
| `POST` | `/api/v1/licensing/plans/{id}/versions` | manage |
| `POST` | `/api/v1/licensing/plan-versions/{id}/publish` | manage |
| `GET` | `/api/v1/licensing/plan-versions/{id}/features` | read |
| `PUT` | `/api/v1/licensing/plan-versions/{id}/features` | manage (draft only) |
| `GET/POST` | `/api/v1/licensing/prices` | catalog |
| `GET/POST` | `/api/v1/licensing/addons` | catalog |
| `POST` | `/api/v1/licensing/addon-versions/{id}/publish` | manage |

Publishing freezes version contents → `409` on further feature edits.

---

## 3. Subscriptions

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/licensing/subscriptions` | `licensing.subscription.manage` |
| `POST` | `/api/v1/licensing/subscriptions` | `licensing.subscription.manage` |
| `GET` | `/api/v1/licensing/subscriptions/{id}` | manage |
| `PATCH` | `/api/v1/licensing/subscriptions/{id}` | manage + `If-Match` |
| `POST` | `/api/v1/licensing/subscriptions/{id}/change-plan` | manage |
| `POST` | `/api/v1/licensing/subscriptions/{id}/pause` | manage |
| `POST` | `/api/v1/licensing/subscriptions/{id}/resume` | manage |
| `POST` | `/api/v1/licensing/subscriptions/{id}/cancel` | manage |
| `GET` | `/api/v1/licensing/subscriptions/{id}/changes` | manage |
| `GET/POST` | `/api/v1/licensing/subscriptions/{id}/schedules` | manage |

**POST create (sketch):**

```json
{
  "plan_version_id": "…",
  "billing_account_id": "…",
  "trial_policy_id": null,
  "items": [{ "price_id": "…", "quantity": 1 }]
}
```

Triggers entitlement compile asynchronously (p14) when `sync_compile=false` → **HTTP 202** + `job_id`. Default `sync_compile=true` stays **200** for tests/budget.

---

## 3.1 Dual house — platform owner vs tenant owner

Same module (`p26_licensing`). Two houses, same JWT. Platform work is never filtered to the login `tenant_id`. Tenant work is always the JWT tenant. Payments stay fail-closed (`PROVIDER_PENDING`).

| House | Who | What they may do |
|---|---|---|
| **Platform** | `platform_admin` / `superadmin` / `licensing.admin` / `licensing.*` | Catalog write + publish. List / assign / pause / resume / cancel / override **any** tenant. Complimentary and trial grants. |
| **Tenant** | Tenant admin with `licensing.subscription.*` | Browse **published** catalog. Subscribe, change-plan, cancel **own tenant only**. Read compiled entitlements. Cannot pause, override, or edit catalog. |

### Platform house

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/licensing/platform/catalog` | `licensing.catalog.read` |
| `POST` | `/api/v1/licensing/platform/catalog/products` | `licensing.catalog.manage` |
| `POST` | `/api/v1/licensing/platform/catalog/plans` | manage |
| `POST` | `/api/v1/licensing/platform/catalog/plans/{id}/versions` | manage |
| `PUT` | `/api/v1/licensing/platform/catalog/plan-versions/{id}/features` | manage (draft only) |
| `POST` | `/api/v1/licensing/platform/catalog/plan-versions/{id}/publish` | manage |
| `POST` | `/api/v1/licensing/platform/catalog/features` | manage |
| `GET` | `/api/v1/licensing/platform/subscriptions?tenant_id=` | `licensing.subscription.read` |
| `POST` | `/api/v1/licensing/platform/subscriptions` | `licensing.subscription.manage` |
| `GET` | `/api/v1/licensing/platform/subscriptions/{id}` | read |
| `POST` | `/api/v1/licensing/platform/subscriptions/{id}/change-plan` | manage |
| `POST` | `/api/v1/licensing/platform/subscriptions/{id}/pause` | manage |
| `POST` | `/api/v1/licensing/platform/subscriptions/{id}/resume` | manage |
| `POST` | `/api/v1/licensing/platform/subscriptions/{id}/cancel` | manage |
| `GET` | `/api/v1/licensing/platform/subscriptions/{id}/entitlement` | `licensing.entitlement.read` |
| `POST` | `/api/v1/licensing/platform/subscriptions/{id}/entitlement/compile` | compile |
| `POST` | `/api/v1/licensing/platform/subscriptions/{id}/entitlement/overrides` | `licensing.admin` |
| `GET` | `/api/v1/licensing/platform/tenants/{tenant_id}/subscription` | read |

**POST assign (sketch):**

```json
{
  "tenant_id": "…",
  "plan_version_id": "…",
  "grant_type": "COMPLIMENTARY",
  "replace": false
}
```

`grant_type`: `PAID` | `TRIAL` | `COMPLIMENTARY`. Live subscription for that tenant → `409 CONFLICT` unless `replace=true` (then change-plan). Tenant callers receive `403 FORBIDDEN`.

### Tenant house

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/licensing/tenant/catalog` | `licensing.catalog.read` (published plans only) |
| `GET` | `/api/v1/licensing/tenant/subscription` | `licensing.subscription.read` (manage implies read) |
| `GET` | `/api/v1/licensing/subscription` | same alias (frontend bootstrap) |
| `POST` | `/api/v1/licensing/tenant/subscription` | `licensing.subscription.manage` |
| `POST` | `/api/v1/licensing/tenant/subscription/change-plan` | manage |
| `POST` | `/api/v1/licensing/tenant/subscription/cancel` | manage |
| `GET` | `/api/v1/licensing/tenant/entitlements` | `licensing.entitlement.read` |
| `POST` | `/api/v1/licensing/tenant/entitlements/check-feature` | read |
| `POST` | `/api/v1/licensing/tenant/entitlements/check-limit` | read |
| `POST` | `/api/v1/licensing/tenant/effective/check` | entitlement.read — server AND p12+p01 |

Tenant subscribe ignores any client `tenant_id`. Complimentary or trial grant from a tenant caller is `403 FORBIDDEN`. Second live subscribe → `409 CONFLICT`.

House **reads and writes prefer Postgres** when the session is a real `AsyncSession`. Empty ledger hydrates once from the seed catalog. TestClient / AsyncMock still uses the in-memory store.

| Method | Path | Notes |
|---|---|---|
| `POST` | `/api/v1/licensing/tenant/payment-intent` | Fail-closed. Live Stripe/Razorpay return `PROVIDER_PENDING` with no invented PSP id. |
| `POST` | `/api/v1/licensing/platform/payment-intent` | Same adapter; platform billing permission. |

---

## 4. Entitlements — compile, check, token

### 4.1 Read / compile

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/licensing/subscriptions/{id}/entitlement` | `licensing.entitlement.read` |
| `POST` | `/api/v1/licensing/subscriptions/{id}/entitlement/compile` | `licensing.entitlement.compile` |
| `POST` | `/api/v1/licensing/subscriptions/{id}/entitlement/overrides` | `licensing.admin` |
| `POST` | `/api/v1/licensing/subscriptions/{id}/entitlement/token` | `licensing.entitlement.read` |

### 4.2 Runtime checks (hot path)

`POST /api/v1/licensing/subscriptions/{id}/entitlement/check-feature`  
**Permission:** `licensing.entitlement.read`  

```json
{ "feature_code": "module.sales.advanced" }
```

```json
{
  "entitled": true,
  "feature_code": "module.sales.advanced",
  "source_type": "PLAN",
  "compiled_at": "…"
}
```

`POST /api/v1/licensing/subscriptions/{id}/entitlement/check-limit`

```json
{ "feature_code": "seats.users", "requested": 1 }
```

```json
{
  "allowed": true,
  "limit": 25,
  "used": 18,
  "remaining": 7
}
```

Over-quota → **429** `QUOTA_EXCEEDED` with the same fields in `data` (`allowed: false`). Dunning **402/423 is not applied** on this hot path until billing lands.

### 4.3 Platform effective gate (SDK)

`POST /api/v1/licensing/effective/check`  
Tenant alias: `POST /api/v1/licensing/tenant/effective/check`

```json
{
  "subscription_id": "…",
  "feature_code": "module.sales.advanced",
  "flag_key": "transport.advanced.ui",
  "permission_code": "transport.advanced.use"
}
```

```json
{
  "entitled": true,
  "flag_on": true,
  "permitted": false,
  "effective": false,
  "feature_code": "module.sales.advanced",
  "note": "server AND: entitled (p26 compiled) AND flag_on (p12 evaluate in-process) AND permitted (p01 codes from JWT)."
}
```

**Server aggregates (this is the SDK).** Callers do not HTTP-loop to p12 or p01.

| Field | Source | Fail-open |
|---|---|---|
| `entitled` | p26 compiled entitlement | no — missing grant is `false` |
| `flag_on` | p12 `evaluate` in-process | omitted / unknown / empty catalog / p12 error → `true` |
| `permitted` | JWT `permissions` (same wildcard rules as the SPA) | omitted `permission_code` → `true`; `is_admin` → `true` |
| `effective` | `entitled AND flag_on AND permitted` | — |

Frontend helper: `isEffectiveCheck` in `mapSubscription.ts`. `licensingApi.checkEffective` posts the tenant alias. Session hydrate / `isFeatureEnabled` stay fail-open until Phase F.

No flag-entitlement compile table is invented. This unblocks **PROD-KERN-012** at the runtime AND, not as a new encyclopedia schema.

### 4.4 Introspect token

`POST /api/v1/licensing/entitlement-tokens/introspect`  
Body: `{ "token": "…" }` → claims + validity (no long-lived secrets in logs).

Compact HMAC JWT (`HS256`), TTL 15 minutes. Unparseable token → **404**. Well-formed unknown / bad signature / expired → **200** `{ valid: false }`. Rotating the current key leaves previous keys active so in-flight tokens still verify (no downtime). Dev secret lives in `secret_ref` (not KMS).

---

## 5. Usage & metering

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/licensing/usage/events` | `licensing.usage.ingest` |
| `POST` | `/api/v1/licensing/usage/events/batch` | ingest |
| `GET` | `/api/v1/licensing/subscriptions/{id}/usage` | `licensing.usage.read` |
| `GET` | `/api/v1/licensing/subscriptions/{id}/usage/ledger` | read |
| `POST` | `/api/v1/licensing/subscriptions/{id}/usage/reset` | admin |
| `POST` | `/api/v1/licensing/subscriptions/{id}/usage/overage/calculate` | manage |
| `POST` | `/api/v1/licensing/usage/rating` | manage |
| `GET/POST` | `/api/v1/licensing/meters` | catalog |

**Ingest:**

`Idempotency-Key` header **or** body `idempotency_key` (header fills when body omits it). Duplicate key → `200` with `duplicate: true`.

```json
{
  "subscription_id": "…",
  "meter_code": "ai.tokens",
  "quantity": 1500,
  "occurred_at": "2026-09-09T05:00:00Z",
  "idempotency_key": "ai-req-…"
}
```

**Overage:** `POST …/usage/overage/calculate` stores `license_usage_overage` (quantity + rated amount). Invoice posting is **Phase G**.

**Clocks (p17):** `licensing.trial.end` (due trial → `EXPIRED`, no auto-charge), `licensing.period.roll` (overage calc + usage reset + 30d window), `licensing.token.expire`. Not RAM timers.

**Outbox / audit / metrics:** commercial writes attach `license_outbox_event` in the same commit and relay fail-open to p13. Catalog publish / assign / cancel / override ingest p19. Compile latency, check counters, and quota breaches write p21 samples (`licensing.runtime` dashboard).

---

## 6. Checkout, quotes, coupons

| Method | Path | Permission |
|---|---|---|
| `POST` | `/api/v1/licensing/checkouts` | `licensing.checkout.manage` |
| `POST` | `/api/v1/licensing/checkouts/{id}/items` | manage |
| `POST` | `/api/v1/licensing/checkouts/{id}/coupons` | manage |
| `POST` | `/api/v1/licensing/checkouts/{id}/recalculate` | manage |
| `POST` | `/api/v1/licensing/checkouts/{id}/confirm` | manage + Idempotency-Key |
| `GET/POST` | `/api/v1/licensing/quotes` | manage |
| `POST` | `/api/v1/licensing/quotes/{id}/convert` | manage |
| `GET/POST` | `/api/v1/licensing/coupons` | admin/manage |

Confirm creates subscription + payment intent **reference** (execution via payment/p23).

---

## 7. Billing coordination & dunning

| Method | Path | Permission |
|---|---|---|
| `GET` | `/api/v1/licensing/invoices` | `licensing.billing.manage` |
| `GET` | `/api/v1/licensing/invoices/{id}` | manage |
| `POST` | `/api/v1/licensing/invoices/{id}/finalize` | manage |
| `POST` | `/api/v1/licensing/invoices/{id}/void` | manage |
| `POST` | `/api/v1/licensing/payment-references` | manage |
| `GET/POST` | `/api/v1/licensing/dunning/policies` | admin |
| `GET` | `/api/v1/licensing/dunning/cases` | manage |
| `POST` | `/api/v1/licensing/dunning/cases/{id}/advance` | manage |

---

## 8. Contracts, marketplace, on-prem

| Method | Path | Permission |
|---|---|---|
| `GET/POST` | `/api/v1/licensing/contracts` | subscription/admin |
| `GET/POST` | `/api/v1/licensing/marketplace/apps` | admin |
| `POST` | `/api/v1/licensing/marketplace/installs` | manage |
| `POST` | `/api/v1/licensing/onprem/licenses` | `licensing.onprem.manage` |
| `POST` | `/api/v1/licensing/onprem/licenses/{id}/activate` | onprem |
| `POST` | `/api/v1/licensing/onprem/licenses/{id}/heartbeat` | onprem |

---

## 9. Webhooks, packs, admin

| Method | Path | Permission |
|---|---|---|
| `GET/POST` | `/api/v1/licensing/webhooks` | admin |
| `GET` | `/api/v1/licensing/packages` | `licensing.admin` |
| `POST` | `/api/v1/licensing/packages/{key}/apply` | admin |
| `POST` | `/api/v1/licensing/signing-keys/rotate` | admin |
| `GET` | `/api/v1/licensing/health/live` | public/internal |
| `GET` | `/api/v1/licensing/health/ready` | internal |

---

## 10. Permission matrix (summary)

| Surface | Min permission |
|---|---|
| Catalog read | `licensing.catalog.read` |
| Catalog write / publish | `licensing.catalog.manage` |
| Subscriptions | `licensing.subscription.read` / `licensing.subscription.manage` |
| Entitlement read/check | `licensing.entitlement.read` |
| Compile | `licensing.entitlement.compile` |
| Usage ingest | `licensing.usage.ingest` |
| Usage read | `licensing.usage.read` |
| Checkout/quotes | `licensing.checkout.manage` |
| Invoices/dunning | `licensing.billing.manage` |
| On-prem | `licensing.onprem.manage` |
| Packs / signing keys | `licensing.admin` |

`licensing.subscription.manage` implies `licensing.subscription.read` (same for catalog manage→read). Logout / login refreshes `GET /api/v1/me/permissions`.

**Seeded role grants** (`seed_and_grant_licensing_permissions` / Alembic `d5e6f718a920`):

| Role | Granted |
|---|---|
| `platform_admin`, `super_admin`, `superadmin`, `superadmin_role` | `licensing.*` |
| `tenant_admin`, `admin` | `licensing.catalog.read`, `licensing.subscription.read`, `licensing.subscription.manage`, `licensing.entitlement.read` |

Tenant roles do **not** receive `licensing.catalog.manage`, `licensing.admin`, or `licensing.*`. Those would open the platform house. Complimentary / trial / pause / override stay platform-only (`403`).

---

## 11. Example flows

### 11.1 Subscribe + enforce

1. `POST /plans/…/publish` (admin)  
2. `POST /checkouts` → confirm → subscription  
3. Compile entitlement (auto)  
4. App: `check-feature` then p12 flag then p01 permission  
5. Meter usage with idempotency keys  

### 11.2 Plan change mid-cycle

1. `POST /subscriptions/{id}/change-plan`  
2. Proration calc stored  
3. Recompile entitlement  
4. Invalidate tokens/cache  

### 11.3 Past due

1. Payment ref fails → dunning case  
2. Soft block → `402`/`423` on checks per policy  
3. Notify via p15; audit via p19  

---

## 12. Related documents

- Guide V2: [`V2_LICENSING_GUIDE.md`](V2_LICENSING_GUIDE.md)  
- Schema V2: [`V2_LICENSING_SCHEMA.md`](V2_LICENSING_SCHEMA.md)  
- Feature: [`../12_feature/FEATURE_API.md`](../12_feature/FEATURE_API.md)  
- API platform: [`../22_api/API_ENDPOINTS.md`](../22_api/API_ENDPOINTS.md)  
- AI: [`../27_ai/AI_API.md`](../27_ai/AI_API.md)  
- v1 full API: [`LICENSING_API.md`](LICENSING_API.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
