# [ARCHIVED] p26_licensing — Complete API Endpoint Specification

> **HYG-015:** This file is the **v1 encyclopedia** (paths, payloads, edge cases). It is **not** the ship REST contract.  
> Canonical HTTP is [`V2_LICENSING_API.md`](V2_LICENSING_API.md) under `/api/v1/licensing`.  
> Bare encyclopedia mounts (`/api/v1/products`, `/api/v1/product-versions`, `/api/v1/product-categories`) return **410** `LICENSING_V1_GONE`.

# p26_licensing — Complete API Endpoint Specification (encyclopedia)

**API Version:** v1  
**Document Version:** 1.2 (last reviewed 2026-09-08)  
**Service / package:** `platforms.p26_licensing`  
**Base Path:** `/api/v1`  
**Protocol:** HTTPS / REST JSON  
**Authentication:** Bearer JWT issued by `p01_identity`  
**Tenant Context:** Derived from verified token claims; never trusted from a client-supplied tenant header.

### Revision history

| Doc version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial complete endpoint specification. |
| 1.1 | 2026-09-08 | Documented optimistic concurrency via `If-Match: <version>` and the 409 version-conflict contract (§1.8), matching the schema's integer `version` (`version_id_col`) and the ORG/Configuration platforms; added platform rate-limiting with the `Retry-After` header (§1.9). |
| 1.2 | 2026-09-08 | Service renamed to `p26_licensing`. Identity → `p01_identity`; tenant master → `p02_organization`. Future integrations use logical names `payment` / `tax` / `accounting` (no platform numbers). See `docs/PLATFORM_REGISTRY.md`. |

---

# 0. Repository Integration Contract

| Item | Frozen value |
|---|---|
| Platform package | `platforms.p26_licensing` |
| Auth issuer | `p01_identity` |
| Tenant/company source | `p02_organization` |
| Payment execution | `payment` *(logical — unnumbered until assigned)* |
| Tax engine | `tax` *(logical)* |
| GL / revenue | `accounting` *(logical)* |
| Numbering authority | [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md) |

---

# 1. API Conventions

## 1.1 Base URL

```text
https://<licensing-host>/api/v1
```

## 1.2 Authentication

```http
Authorization: Bearer <access_token>
```

The access token should contain the authenticated subject and authorized tenant context.

## 1.3 Request Correlation

Recommended headers:

```http
Authorization: Bearer <token>
X-Request-ID: <request-id>
X-Correlation-ID: <correlation-id>
Idempotency-Key: <unique-key>
If-Match: <version>
```

`Idempotency-Key` is required for commands where duplicate execution could create duplicate commercial state.

`If-Match` carries the integer optimistic-lock `version` on concurrency-sensitive updates — see §1.8.

## 1.4 Content Type

```http
Content-Type: application/json
Accept: application/json
```

## 1.5 Resource IDs

Use UUID/ULID-style opaque identifiers.

Do not expose sequential database IDs.

## 1.6 Pagination

Collection endpoints should support:

```text
page
page_size
cursor
limit
```

Preferred production approach for large collections:

```text
cursor-based pagination
```

Example:

```http
GET /api/v1/subscriptions?cursor=<cursor>&limit=50
```

## 1.7 Standard Response Envelope

Successful single resource:

```json
{
  "data": {},
  "meta": {
    "request_id": "req_123"
  }
}
```

Collection:

```json
{
  "data": [],
  "meta": {
    "request_id": "req_123",
    "next_cursor": "..."
  }
}
```

## 1.8 Optimistic Concurrency

Mutable resources carry an integer `version` (the schema's `version_id_col`, consistent with the ORG and Configuration platforms). Concurrency-sensitive updates — `PATCH` on drafts, quantity/plan changes, quota-affecting operations — must send the version the client last read, either as an `If-Match` header or an `expected_version` body field (both carry the same value):

```http
PATCH /api/v1/plans/{plan_id}
If-Match: 4
```

or:

```json
{ "expected_version": 4 }
```

If the stored `version` has advanced, the server returns `409` and does not apply the change:

```json
{
  "error": {
    "code": "VERSION_CONFLICT",
    "message": "The resource was modified by another request.",
    "details": { "expected_version": 4, "current_version": 5 },
    "request_id": "req_123",
    "correlation_id": "corr_123"
  }
}
```

The `version` is returned on every mutable resource so clients can round-trip it. (`row_version`/change tokens are internal and are never the `If-Match` value.) Never silently overwrite a concurrent change.

## 1.9 Rate Limiting

Public APIs are rate limited. When a caller exceeds its quota the API returns `429` with a `Retry-After` header (seconds):

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

Limits are tiered — higher for reads, lower for writes, and very low for sensitive operations (signing-key operations, license revocation, entitlement overrides, event/outbox replay). This is the API transport limit; it is distinct from the entitlement **rate-limit policies** a subscription is granted (§44).

---

# 2. Standard Error Response

```json
{
  "error": {
    "code": "SUBSCRIPTION_NOT_FOUND",
    "message": "Subscription was not found.",
    "details": {},
    "request_id": "req_123",
    "correlation_id": "corr_123"
  }
}
```

Common HTTP statuses:

| Status | Meaning |
|---:|---|
| 200 | Successful request |
| 201 | Resource created |
| 202 | Accepted for asynchronous processing |
| 204 | Successful request without response body |
| 400 | Invalid request |
| 401 | Authentication required/invalid |
| 403 | Permission denied |
| 404 | Resource not found |
| 409 | Conflict |
| 422 | Business validation failure |
| 429 | Rate limit exceeded |
| 500 | Internal error |
| 503 | Service unavailable |

---

# 3. API Domains

```text
1. Health / Metadata
2. Catalog
3. Products
4. Product Versions
5. Features
6. Modules
7. Units
8. Currencies
9. Plans
10. Plan Versions
11. Pricing
12. Add-ons
13. Subscriptions
14. Subscription Items
15. Subscription Changes
16. Subscription Schedules
17. Trials
18. Contracts
19. Checkout
20. Quotes
21. Discounts
22. Coupons
23. Entitlements
24. Entitlement Compilation
25. Signing Keys
26. Meters
27. Usage
28. Usage Periods
29. Usage Rating / Overage
30. Rate Limits
31. Commercial Billing
32. Proration
33. Dunning
34. Payment References
35. Refund / Chargeback References
36. Marketplace
37. On-Premise Licensing
38. Webhooks
39. Events
40. Idempotency
41. Audit
42. Configuration
43. Administrative Operations
```

---

# 4. Health & Metadata APIs

## 4.1 Health

```http
GET /health
```

Purpose:

```text
Liveness check
```

## 4.2 Readiness

```http
GET /ready
```

Checks:

```text
PostgreSQL
Redis
Message Broker
Required External Dependencies
```

## 4.3 Service Metadata

```http
GET /api/v1/metadata
```

Returns:

```text
service_name
version
api_version
environment
capabilities
```

---

# 5. Product Catalog APIs

## 5.1 Create Product

```http
POST /api/v1/products
```

Request:

```json
{
  "code": "NEXTERP",
  "name": "NEXTERP",
  "description": "Enterprise ERP platform",
  "product_type": "PLATFORM",
  "status": "ACTIVE"
}
```

Response:

```text
201 Created
```

---

## 5.2 List Products

```http
GET /api/v1/products
```

Filters:

```text
status
product_type
category_id
search
```

---

## 5.3 Get Product

```http
GET /api/v1/products/{product_id}
```

---

## 5.4 Update Product

```http
PATCH /api/v1/products/{product_id}
```

Use only for mutable metadata.

Do not mutate historical commercial meaning.

---

## 5.5 Archive Product

```http
POST /api/v1/products/{product_id}/archive
```

---

## 5.6 Restore Product

```http
POST /api/v1/products/{product_id}/restore
```

---

# 6. Product Version APIs

## 6.1 Create Product Version

```http
POST /api/v1/products/{product_id}/versions
```

## 6.2 List Versions

```http
GET /api/v1/products/{product_id}/versions
```

## 6.3 Get Version

```http
GET /api/v1/product-versions/{product_version_id}
```

## 6.4 Publish Version

```http
POST /api/v1/product-versions/{product_version_id}/publish
```

## 6.5 Retire Version

```http
POST /api/v1/product-versions/{product_version_id}/retire
```

Published versions should be treated as immutable.

---

# 7. Product Category APIs

## 7.1 Create Category

```http
POST /api/v1/product-categories
```

## 7.2 List Categories

```http
GET /api/v1/product-categories
```

## 7.3 Get Category

```http
GET /api/v1/product-categories/{category_id}
```

## 7.4 Update Category

```http
PATCH /api/v1/product-categories/{category_id}
```

## 7.5 Delete / Archive Category

```http
POST /api/v1/product-categories/{category_id}/archive
```

---

# 8. Feature APIs

## 8.1 Create Feature

```http
POST /api/v1/features
```

Request:

```json
{
  "code": "FINANCE",
  "name": "Finance",
  "feature_type": "MODULE"
}
```

Feature types:

```text
BOOLEAN
QUOTA
LIMIT
RATE_LIMIT
METER
ACTION
MODULE
API
```

## 8.2 List Features

```http
GET /api/v1/features
```

Filters:

```text
feature_type
status
product_id
search
```

## 8.3 Get Feature

```http
GET /api/v1/features/{feature_id}
```

## 8.4 Update Feature

```http
PATCH /api/v1/features/{feature_id}
```

## 8.5 Archive Feature

```http
POST /api/v1/features/{feature_id}/archive
```

---

# 9. Feature Dependency APIs

## 9.1 Create Dependency

```http
POST /api/v1/features/{feature_id}/dependencies
```

## 9.2 List Dependencies

```http
GET /api/v1/features/{feature_id}/dependencies
```

## 9.3 Remove Dependency

```http
DELETE /api/v1/features/{feature_id}/dependencies/{dependency_id}
```

Dependency changes should be blocked when they would invalidate an already-published commercial version.

---

# 10. Module APIs

## 10.1 Create Module

```http
POST /api/v1/modules
```

## 10.2 List Modules

```http
GET /api/v1/modules
```

## 10.3 Get Module

```http
GET /api/v1/modules/{module_id}
```

## 10.4 Update Module

```http
PATCH /api/v1/modules/{module_id}
```

## 10.5 Archive Module

```http
POST /api/v1/modules/{module_id}/archive
```

---

# 11. Unit APIs

## 11.1 Create Unit

```http
POST /api/v1/units
```

## 11.2 List Units

```http
GET /api/v1/units
```

## 11.3 Get Unit

```http
GET /api/v1/units/{unit_id}
```

## 11.4 Update Unit

```http
PATCH /api/v1/units/{unit_id}
```

---

# 12. Currency APIs

## 12.1 List Currencies

```http
GET /api/v1/currencies
```

Filters:

```text
active
region
```

## 12.2 Get Currency

```http
GET /api/v1/currencies/{currency_code}
```

Currency metadata should support:

```text
ISO 4217 code
numeric code
symbol
minor units
cash rounding increment
rounding mode
```

---

# 13. Plan APIs

## 13.1 Create Plan

```http
POST /api/v1/plans
```

Request:

```json
{
  "product_id": "uuid",
  "code": "ENTERPRISE",
  "name": "Enterprise",
  "plan_type": "ENTERPRISE"
}
```

## 13.2 List Plans

```http
GET /api/v1/plans
```

Filters:

```text
product_id
plan_type
status
search
```

## 13.3 Get Plan

```http
GET /api/v1/plans/{plan_id}
```

## 13.4 Update Plan Metadata

```http
PATCH /api/v1/plans/{plan_id}
```

## 13.5 Archive Plan

```http
POST /api/v1/plans/{plan_id}/archive
```

---

# 14. Plan Version APIs

## 14.1 Create Plan Version

```http
POST /api/v1/plans/{plan_id}/versions
```

## 14.2 List Plan Versions

```http
GET /api/v1/plans/{plan_id}/versions
```

## 14.3 Get Plan Version

```http
GET /api/v1/plan-versions/{plan_version_id}
```

## 14.4 Add Feature to Plan Version

```http
POST /api/v1/plan-versions/{plan_version_id}/features
```

## 14.5 List Plan Version Features

```http
GET /api/v1/plan-versions/{plan_version_id}/features
```

## 14.6 Update Draft Feature

```http
PATCH /api/v1/plan-versions/{plan_version_id}/features/{feature_id}
```

## 14.7 Remove Draft Feature

```http
DELETE /api/v1/plan-versions/{plan_version_id}/features/{feature_id}
```

## 14.8 Publish Plan Version

```http
POST /api/v1/plan-versions/{plan_version_id}/publish
```

## 14.9 Retire Plan Version

```http
POST /api/v1/plan-versions/{plan_version_id}/retire
```

---

# 15. Pricing APIs

## 15.1 Create Price

```http
POST /api/v1/prices
```

Example:

```json
{
  "plan_version_id": "uuid",
  "currency": "USD",
  "pricing_model": "FLAT",
  "amount": "299.00",
  "billing_interval": "MONTH"
}
```

## 15.2 List Prices

```http
GET /api/v1/prices
```

Filters:

```text
plan_version_id
addon_version_id
currency
pricing_model
billing_interval
status
```

## 15.3 Get Price

```http
GET /api/v1/prices/{price_id}
```

## 15.4 Update Draft Price

```http
PATCH /api/v1/prices/{price_id}
```

## 15.5 Activate Price

```http
POST /api/v1/prices/{price_id}/activate
```

## 15.6 Retire Price

```http
POST /api/v1/prices/{price_id}/retire
```

---

# 16. Price Tier APIs

## 16.1 Create Tier

```http
POST /api/v1/prices/{price_id}/tiers
```

## 16.2 List Tiers

```http
GET /api/v1/prices/{price_id}/tiers
```

## 16.3 Update Tier

```http
PATCH /api/v1/prices/{price_id}/tiers/{tier_id}
```

## 16.4 Delete Draft Tier

```http
DELETE /api/v1/prices/{price_id}/tiers/{tier_id}
```

---

# 17. Price Component APIs

For hybrid prices:

```http
POST /api/v1/prices/{price_id}/components
```

```http
GET /api/v1/prices/{price_id}/components
```

```http
PATCH /api/v1/prices/{price_id}/components/{component_id}
```

```http
DELETE /api/v1/prices/{price_id}/components/{component_id}
```

---

# 18. Add-on APIs

## 18.1 Create Add-on

```http
POST /api/v1/addons
```

## 18.2 List Add-ons

```http
GET /api/v1/addons
```

## 18.3 Get Add-on

```http
GET /api/v1/addons/{addon_id}
```

## 18.4 Update Add-on

```http
PATCH /api/v1/addons/{addon_id}
```

## 18.5 Archive Add-on

```http
POST /api/v1/addons/{addon_id}/archive
```

---

# 19. Add-on Version APIs

```http
POST /api/v1/addons/{addon_id}/versions
```

```http
GET /api/v1/addons/{addon_id}/versions
```

```http
GET /api/v1/addon-versions/{addon_version_id}
```

```http
POST /api/v1/addon-versions/{addon_version_id}/publish
```

```http
POST /api/v1/addon-versions/{addon_version_id}/retire
```

---

# 20. Add-on Feature APIs

```http
POST /api/v1/addon-versions/{addon_version_id}/features
```

```http
GET /api/v1/addon-versions/{addon_version_id}/features
```

```http
PATCH /api/v1/addon-versions/{addon_version_id}/features/{feature_id}
```

```http
DELETE /api/v1/addon-versions/{addon_version_id}/features/{feature_id}
```

---

# 21. Subscription APIs

## 21.1 Create Subscription

```http
POST /api/v1/subscriptions
```

Example:

```json
{
  "customer_id": "customer_uuid",
  "product_id": "product_uuid",
  "plan_version_id": "plan_version_uuid",
  "currency": "USD",
  "billing_interval": "MONTH",
  "quantity": 1,
  "auto_renew": true
}
```

For payment-required subscriptions, creation may return:

```text
202 Accepted
```

when asynchronous payment confirmation is required.

## 21.2 List Subscriptions

```http
GET /api/v1/subscriptions
```

Filters:

```text
customer_id
status
product_id
plan_id
plan_version_id
currency
billing_interval
starts_at
ends_at
```

## 21.3 Get Subscription

```http
GET /api/v1/subscriptions/{subscription_id}
```

## 21.4 Update Subscription Metadata

```http
PATCH /api/v1/subscriptions/{subscription_id}
```

Do not use generic status mutation.

---

# 22. Subscription Lifecycle Commands

## 22.1 Activate

```http
POST /api/v1/subscriptions/{subscription_id}/activate
```

## 22.2 Pause

```http
POST /api/v1/subscriptions/{subscription_id}/pause
```

## 22.3 Resume

```http
POST /api/v1/subscriptions/{subscription_id}/resume
```

## 22.4 Cancel

```http
POST /api/v1/subscriptions/{subscription_id}/cancel
```

Request:

```json
{
  "effective_type": "END_OF_TERM",
  "reason": "CUSTOMER_REQUEST"
}
```

## 22.5 Reactivate

```http
POST /api/v1/subscriptions/{subscription_id}/reactivate
```

## 22.6 Expire

```http
POST /api/v1/subscriptions/{subscription_id}/expire
```

Normally worker/admin controlled.

---

# 23. Subscription Item APIs

## 23.1 List Items

```http
GET /api/v1/subscriptions/{subscription_id}/items
```

## 23.2 Add Item

```http
POST /api/v1/subscriptions/{subscription_id}/items
```

## 23.3 Get Item

```http
GET /api/v1/subscription-items/{item_id}
```

## 23.4 Change Quantity

```http
POST /api/v1/subscription-items/{item_id}/change-quantity
```

## 23.5 Remove Item

```http
POST /api/v1/subscription-items/{item_id}/remove
```

---

# 24. Subscription Plan Change APIs

## 24.1 Preview Plan Change

```http
POST /api/v1/subscriptions/{subscription_id}/change-plan/preview
```

Returns:

```text
Current Plan
New Plan
Effective Date
Unused Credit
New Charge
Proration
Tax Reference
Net Amount
```

## 24.2 Execute Plan Change

```http
POST /api/v1/subscriptions/{subscription_id}/change-plan
```

Request:

```json
{
  "new_plan_version_id": "uuid",
  "effective_type": "IMMEDIATE",
  "proration": true
}
```

---

# 25. Subscription Change APIs

## 25.1 List Changes

```http
GET /api/v1/subscriptions/{subscription_id}/changes
```

## 25.2 Get Change

```http
GET /api/v1/subscription-changes/{change_id}
```

## 25.3 Apply Scheduled Change

```http
POST /api/v1/subscription-changes/{change_id}/apply
```

## 25.4 Cancel Scheduled Change

```http
POST /api/v1/subscription-changes/{change_id}/cancel
```

---

# 26. Subscription Schedule APIs

## 26.1 Create Schedule

```http
POST /api/v1/subscriptions/{subscription_id}/schedules
```

## 26.2 List Schedules

```http
GET /api/v1/subscriptions/{subscription_id}/schedules
```

## 26.3 Get Schedule

```http
GET /api/v1/subscription-schedules/{schedule_id}
```

## 26.4 Add Phase

```http
POST /api/v1/subscription-schedules/{schedule_id}/phases
```

## 26.5 List Phases

```http
GET /api/v1/subscription-schedules/{schedule_id}/phases
```

## 26.6 Update Draft Phase

```http
PATCH /api/v1/subscription-schedule-phases/{phase_id}
```

## 26.7 Activate Schedule

```http
POST /api/v1/subscription-schedules/{schedule_id}/activate
```

## 26.8 Cancel Schedule

```http
POST /api/v1/subscription-schedules/{schedule_id}/cancel
```

---

# 27. Trial APIs

## 27.1 Create Trial Policy

```http
POST /api/v1/trial-policies
```

## 27.2 List Trial Policies

```http
GET /api/v1/trial-policies
```

## 27.3 Get Trial Policy

```http
GET /api/v1/trial-policies/{policy_id}
```

## 27.4 Update Trial Policy

```http
PATCH /api/v1/trial-policies/{policy_id}
```

## 27.5 Start Trial

```http
POST /api/v1/subscriptions/{subscription_id}/trial
```

## 27.6 Get Subscription Trial

```http
GET /api/v1/subscriptions/{subscription_id}/trial
```

## 27.7 Extend Trial

```http
POST /api/v1/subscriptions/{subscription_id}/trial/extend
```

## 27.8 End Trial

```http
POST /api/v1/subscriptions/{subscription_id}/trial/end
```

---

# 28. Contract APIs

## 28.1 Create Contract

```http
POST /api/v1/contracts
```

## 28.2 List Contracts

```http
GET /api/v1/contracts
```

Filters:

```text
customer_id
status
start_date
end_date
```

## 28.3 Get Contract

```http
GET /api/v1/contracts/{contract_id}
```

## 28.4 Update Draft Contract

```http
PATCH /api/v1/contracts/{contract_id}
```

## 28.5 Activate Contract

```http
POST /api/v1/contracts/{contract_id}/activate
```

## 28.6 Renew Contract

```http
POST /api/v1/contracts/{contract_id}/renew
```

## 28.7 Terminate Contract

```http
POST /api/v1/contracts/{contract_id}/terminate
```

---

# 29. Contract Line APIs

```http
POST /api/v1/contracts/{contract_id}/lines
```

```http
GET /api/v1/contracts/{contract_id}/lines
```

```http
PATCH /api/v1/contract-lines/{line_id}
```

```http
DELETE /api/v1/contract-lines/{line_id}
```

---

# 30. Checkout APIs

## 30.1 Create Checkout

```http
POST /api/v1/checkouts
```

## 30.2 Get Checkout

```http
GET /api/v1/checkouts/{checkout_id}
```

## 30.3 Update Checkout

```http
PATCH /api/v1/checkouts/{checkout_id}
```

## 30.4 Add Checkout Item

```http
POST /api/v1/checkouts/{checkout_id}/items
```

## 30.5 List Checkout Items

```http
GET /api/v1/checkouts/{checkout_id}/items
```

## 30.6 Remove Checkout Item

```http
DELETE /api/v1/checkouts/{checkout_id}/items/{item_id}
```

## 30.7 Apply Coupon

```http
POST /api/v1/checkouts/{checkout_id}/coupons
```

## 30.8 Remove Coupon

```http
DELETE /api/v1/checkouts/{checkout_id}/coupons/{coupon_id}
```

## 30.9 Recalculate Checkout

```http
POST /api/v1/checkouts/{checkout_id}/recalculate
```

## 30.10 Confirm Checkout

```http
POST /api/v1/checkouts/{checkout_id}/confirm
```

## 30.11 Cancel Checkout

```http
POST /api/v1/checkouts/{checkout_id}/cancel
```

---

# 31. Quote APIs

## 31.1 Create Quote

```http
POST /api/v1/quotes
```

## 31.2 List Quotes

```http
GET /api/v1/quotes
```

## 31.3 Get Quote

```http
GET /api/v1/quotes/{quote_id}
```

## 31.4 Update Draft Quote

```http
PATCH /api/v1/quotes/{quote_id}
```

## 31.5 Add Quote Line

```http
POST /api/v1/quotes/{quote_id}/lines
```

## 31.6 List Quote Lines

```http
GET /api/v1/quotes/{quote_id}/lines
```

## 31.7 Update Quote Line

```http
PATCH /api/v1/quote-lines/{line_id}
```

## 31.8 Remove Quote Line

```http
DELETE /api/v1/quote-lines/{line_id}
```

## 31.9 Send Quote

```http
POST /api/v1/quotes/{quote_id}/send
```

## 31.10 Accept Quote

```http
POST /api/v1/quotes/{quote_id}/accept
```

## 31.11 Reject Quote

```http
POST /api/v1/quotes/{quote_id}/reject
```

## 31.12 Convert Quote to Subscription

```http
POST /api/v1/quotes/{quote_id}/convert-to-subscription
```

---

# 32. Discount APIs

## 32.1 Create Discount

```http
POST /api/v1/discounts
```

## 32.2 List Discounts

```http
GET /api/v1/discounts
```

## 32.3 Get Discount

```http
GET /api/v1/discounts/{discount_id}
```

## 32.4 Update Discount

```http
PATCH /api/v1/discounts/{discount_id}
```

## 32.5 Activate Discount

```http
POST /api/v1/discounts/{discount_id}/activate
```

## 32.6 Deactivate Discount

```http
POST /api/v1/discounts/{discount_id}/deactivate
```

---

# 33. Coupon APIs

## 33.1 Create Coupon

```http
POST /api/v1/coupons
```

## 33.2 List Coupons

```http
GET /api/v1/coupons
```

## 33.3 Get Coupon

```http
GET /api/v1/coupons/{coupon_id}
```

## 33.4 Update Draft Coupon

```http
PATCH /api/v1/coupons/{coupon_id}
```

## 33.5 Activate Coupon

```http
POST /api/v1/coupons/{coupon_id}/activate
```

## 33.6 Deactivate Coupon

```http
POST /api/v1/coupons/{coupon_id}/deactivate
```

## 33.7 Validate Coupon

```http
POST /api/v1/coupons/validate
```

Request:

```json
{
  "code": "STARTUP2026",
  "customer_id": "uuid",
  "plan_version_id": "uuid",
  "currency": "USD"
}
```

## 33.8 List Coupon Usage

```http
GET /api/v1/coupons/{coupon_id}/usage
```

---

# 34. Entitlement APIs

## 34.1 Get Current Entitlement

```http
GET /api/v1/subscriptions/{subscription_id}/entitlement
```

## 34.2 Get Effective Features

```http
GET /api/v1/subscriptions/{subscription_id}/entitlement/features
```

## 34.3 Get Effective Limits

```http
GET /api/v1/subscriptions/{subscription_id}/entitlement/limits
```

## 34.4 Check Feature

```http
POST /api/v1/subscriptions/{subscription_id}/entitlement/check-feature
```

Request:

```json
{
  "feature_code": "FINANCE"
}
```

Response:

```json
{
  "data": {
    "allowed": true
  }
}
```

## 34.5 Check Limit

```http
POST /api/v1/subscriptions/{subscription_id}/entitlement/check-limit
```

Request:

```json
{
  "feature_code": "MAX_USERS",
  "requested": 5
}
```

---

# 35. Entitlement Override APIs

Administrative overrides must be strongly authorized.

## 35.1 Create Override

```http
POST /api/v1/subscriptions/{subscription_id}/entitlement-overrides
```

## 35.2 List Overrides

```http
GET /api/v1/subscriptions/{subscription_id}/entitlement-overrides
```

## 35.3 Get Override

```http
GET /api/v1/entitlement-overrides/{override_id}
```

## 35.4 Update Draft Override

```http
PATCH /api/v1/entitlement-overrides/{override_id}
```

## 35.5 Activate Override

```http
POST /api/v1/entitlement-overrides/{override_id}/activate
```

## 35.6 Revoke Override

```http
POST /api/v1/entitlement-overrides/{override_id}/revoke
```

Every override should record:

```text
actor
reason
source
effective_at
expiration
```

---

# 36. Entitlement Compilation APIs

## 36.1 Request Compilation

```http
POST /api/v1/subscriptions/{subscription_id}/entitlement/compile
```

Normally returns:

```text
202 Accepted
```

## 36.2 Get Compilation

```http
GET /api/v1/entitlement-compilations/{compilation_id}
```

## 36.3 Get Compilation History

```http
GET /api/v1/subscriptions/{subscription_id}/entitlement/compilations
```

## 36.4 Force Recompile

```http
POST /api/v1/subscriptions/{subscription_id}/entitlement/recompile
```

Administrative endpoint.

---

# 37. Entitlement Token APIs

## 37.1 Get Current Token

```http
GET /api/v1/subscriptions/{subscription_id}/entitlement/token
```

## 37.2 Refresh Token

```http
POST /api/v1/subscriptions/{subscription_id}/entitlement/token/refresh
```

## 37.3 Revoke Token

```http
POST /api/v1/subscriptions/{subscription_id}/entitlement/token/revoke
```

The runtime token should be short-lived.

---

# 38. Signing Key APIs

These are platform-admin APIs.

## 38.1 List Keys

```http
GET /api/v1/admin/signing-keys
```

## 38.2 Get Key Metadata

```http
GET /api/v1/admin/signing-keys/{key_id}
```

## 38.3 Generate Key

```http
POST /api/v1/admin/signing-keys
```

## 38.4 Activate Key

```http
POST /api/v1/admin/signing-keys/{key_id}/activate
```

## 38.5 Retire Key

```http
POST /api/v1/admin/signing-keys/{key_id}/retire
```

## 38.6 Revoke Key

```http
POST /api/v1/admin/signing-keys/{key_id}/revoke
```

Private key material should remain in KMS/HSM/secret infrastructure.

---

# 39. Meter APIs

## 39.1 Create Meter

```http
POST /api/v1/meters
```

Example:

```json
{
  "code": "API_REQUESTS",
  "name": "API Requests",
  "aggregation_type": "COUNT",
  "unit": "REQUEST"
}
```

## 39.2 List Meters

```http
GET /api/v1/meters
```

## 39.3 Get Meter

```http
GET /api/v1/meters/{meter_id}
```

## 39.4 Update Meter

```http
PATCH /api/v1/meters/{meter_id}
```

## 39.5 Activate Meter

```http
POST /api/v1/meters/{meter_id}/activate
```

## 39.6 Retire Meter

```http
POST /api/v1/meters/{meter_id}/retire
```

---

# 40. Usage APIs

## 40.1 Record Usage Event

```http
POST /api/v1/usage/events
```

Request:

```json
{
  "idempotency_key": "invoice-created-001",
  "tenant_id": "logical-context-only",
  "subscription_id": "uuid",
  "meter_id": "uuid",
  "quantity": "1",
  "occurred_at": "2026-09-08T10:00:00Z",
  "source": "erp.invoice"
}
```

The authoritative tenant must come from the security/service context, not the request body.

## 40.2 Batch Usage Ingestion

```http
POST /api/v1/usage/events/batch
```

Use bounded batch sizes.

## 40.3 Get Usage

```http
GET /api/v1/subscriptions/{subscription_id}/usage
```

Filters:

```text
meter_id
period_start
period_end
```

## 40.4 Get Usage Event

```http
GET /api/v1/usage/events/{event_id}
```

## 40.5 Get Usage Ledger

```http
GET /api/v1/subscriptions/{subscription_id}/usage/ledger
```

---

# 41. Usage Period APIs

## 41.1 Get Current Usage Period

```http
GET /api/v1/subscriptions/{subscription_id}/usage-periods/current
```

## 41.2 List Usage Periods

```http
GET /api/v1/subscriptions/{subscription_id}/usage-periods
```

## 41.3 Get Usage Period

```http
GET /api/v1/usage-periods/{period_id}
```

## 41.4 Close Usage Period

```http
POST /api/v1/usage-periods/{period_id}/close
```

Worker/admin controlled.

## 41.5 Reset Usage

```http
POST /api/v1/subscriptions/{subscription_id}/usage/reset
```

Reset must create a durable reset ledger entry rather than deleting historical usage.

---

# 42. Usage Overage APIs

## 42.1 List Overage

```http
GET /api/v1/subscriptions/{subscription_id}/usage/overage
```

## 42.2 Get Overage

```http
GET /api/v1/usage/overage/{overage_id}
```

## 42.3 Calculate Overage

```http
POST /api/v1/subscriptions/{subscription_id}/usage/overage/calculate
```

---

# 43. Usage Rating APIs

## 43.1 Rate Usage

```http
POST /api/v1/usage/rating
```

## 43.2 Get Rating

```http
GET /api/v1/usage/ratings/{rating_id}
```

## 43.3 List Ratings

```http
GET /api/v1/subscriptions/{subscription_id}/usage/ratings
```

Rated amounts should preserve the price/calculation version used.

---

# 44. Rate Limit APIs

## 44.1 Create Rate Limit Policy

```http
POST /api/v1/rate-limit-policies
```

Supported algorithms:

```text
TOKEN_BUCKET
SLIDING_WINDOW
FIXED_WINDOW
LEAKY_BUCKET
```

## 44.2 List Policies

```http
GET /api/v1/rate-limit-policies
```

## 44.3 Get Policy

```http
GET /api/v1/rate-limit-policies/{policy_id}
```

## 44.4 Update Policy

```http
PATCH /api/v1/rate-limit-policies/{policy_id}
```

## 44.5 Activate Policy

```http
POST /api/v1/rate-limit-policies/{policy_id}/activate
```

## 44.6 Deactivate Policy

```http
POST /api/v1/rate-limit-policies/{policy_id}/deactivate
```

---

# 45. Commercial Billing APIs

These APIs manage commercial subscription billing state. They do not replace `accounting`.

## 45.1 Create Billing Account

```http
POST /api/v1/billing/accounts
```

## 45.2 Get Billing Account

```http
GET /api/v1/billing/accounts/{billing_account_id}
```

## 45.3 Update Billing Account

```http
PATCH /api/v1/billing/accounts/{billing_account_id}
```

## 45.4 List Billing Accounts

```http
GET /api/v1/billing/accounts
```

---

# 46. Billing Cycle APIs

## 46.1 Get Current Cycle

```http
GET /api/v1/subscriptions/{subscription_id}/billing-cycles/current
```

## 46.2 List Cycles

```http
GET /api/v1/subscriptions/{subscription_id}/billing-cycles
```

## 46.3 Get Cycle

```http
GET /api/v1/billing-cycles/{billing_cycle_id}
```

## 46.4 Close Cycle

```http
POST /api/v1/billing-cycles/{billing_cycle_id}/close
```

## 46.5 Generate Commercial Billing Document

```http
POST /api/v1/billing-cycles/{billing_cycle_id}/generate-document
```

---

# 47. Commercial Invoice APIs

These are subscription-commercial billing documents. Financial accounting remains outside this service.

## 47.1 Create Draft

```http
POST /api/v1/billing/invoices
```

## 47.2 List Invoices

```http
GET /api/v1/billing/invoices
```

Filters:

```text
customer_id
subscription_id
status
currency
issue_date
due_date
```

## 47.3 Get Invoice

```http
GET /api/v1/billing/invoices/{invoice_id}
```

## 47.4 Get Invoice Lines

```http
GET /api/v1/billing/invoices/{invoice_id}/lines
```

## 47.5 Finalize Invoice

```http
POST /api/v1/billing/invoices/{invoice_id}/finalize
```

After finalization, commercial values should be immutable.

## 47.6 Void Invoice

```http
POST /api/v1/billing/invoices/{invoice_id}/void
```

## 47.7 Download Invoice Representation

```http
GET /api/v1/billing/invoices/{invoice_id}/document
```

If PDF generation is delegated to another document platform, return the external document reference instead.

---

# 48. Invoice Line APIs

## 48.1 Add Draft Line

```http
POST /api/v1/billing/invoices/{invoice_id}/lines
```

## 48.2 Update Draft Line

```http
PATCH /api/v1/billing/invoice-lines/{line_id}
```

## 48.3 Remove Draft Line

```http
DELETE /api/v1/billing/invoice-lines/{line_id}
```

Only draft documents may have editable lines.

---

# 49. Customer Balance APIs

## 49.1 Get Balance

```http
GET /api/v1/billing/customers/{customer_id}/balance
```

## 49.2 List Balance Transactions

```http
GET /api/v1/billing/customers/{customer_id}/balance/transactions
```

## 49.3 Create Commercial Balance Adjustment

```http
POST /api/v1/billing/customers/{customer_id}/balance/adjustments
```

Financial accounting impact must be handled by `accounting`.

---

# 50. Proration APIs

## 50.1 Preview Proration

```http
POST /api/v1/proration/preview
```

Request:

```json
{
  "subscription_id": "uuid",
  "new_plan_version_id": "uuid",
  "effective_at": "2026-09-20T00:00:00Z"
}
```

## 50.2 Calculate Proration

```http
POST /api/v1/proration/calculate
```

## 50.3 Get Proration

```http
GET /api/v1/proration/{proration_id}
```

---

# 51. Dunning APIs

## 51.1 Create Dunning Policy

```http
POST /api/v1/dunning/policies
```

## 51.2 List Policies

```http
GET /api/v1/dunning/policies
```

## 51.3 Get Policy

```http
GET /api/v1/dunning/policies/{policy_id}
```

## 51.4 Update Policy

```http
PATCH /api/v1/dunning/policies/{policy_id}
```

## 51.5 List Dunning Cases

```http
GET /api/v1/dunning/cases
```

## 51.6 Get Dunning Case

```http
GET /api/v1/dunning/cases/{case_id}
```

## 51.7 Retry Payment

```http
POST /api/v1/dunning/cases/{case_id}/retry-payment
```

This sends a request to `payment`; licensing does not execute card processing itself.

## 51.8 Suspend

```http
POST /api/v1/dunning/cases/{case_id}/suspend
```

## 51.9 Resolve

```http
POST /api/v1/dunning/cases/{case_id}/resolve
```

---

# 52. Payment Reference APIs

Payment execution belongs to `payment`.

Licensing stores only the integration reference required for subscription state.

## 52.1 List Payment References

```http
GET /api/v1/billing/payment-references
```

## 52.2 Get Payment Reference

```http
GET /api/v1/billing/payment-references/{payment_reference_id}
```

## 52.3 Create Payment Request

```http
POST /api/v1/billing/payment-requests
```

This creates a request/event for `payment`.

## 52.4 Record Payment Result

```http
POST /api/v1/billing/payment-references
```

Normally called by a trusted service integration rather than a public browser client.

---

# 53. Refund Reference APIs

Refund processing belongs to `payment`.

## 53.1 Request Refund

```http
POST /api/v1/billing/refunds
```

## 53.2 Get Refund

```http
GET /api/v1/billing/refunds/{refund_id}
```

## 53.3 List Refunds

```http
GET /api/v1/billing/refunds
```

---

# 54. Chargeback APIs

## 54.1 Record Chargeback

```http
POST /api/v1/billing/chargebacks
```

## 54.2 Get Chargeback

```http
GET /api/v1/billing/chargebacks/{chargeback_id}
```

## 54.3 List Chargebacks

```http
GET /api/v1/billing/chargebacks
```

---

# 55. External Reference APIs

Used for cross-platform references.

## 55.1 Create External Reference

```http
POST /api/v1/external-references
```

## 55.2 List External References

```http
GET /api/v1/external-references
```

## 55.3 Get External Reference

```http
GET /api/v1/external-references/{reference_id}
```

---

# 56. Reconciliation APIs

This is integration reconciliation, not financial accounting reconciliation.

## 56.1 Start Reconciliation

```http
POST /api/v1/reconciliation/runs
```

## 56.2 List Runs

```http
GET /api/v1/reconciliation/runs
```

## 56.3 Get Run

```http
GET /api/v1/reconciliation/runs/{run_id}
```

## 56.4 List Items

```http
GET /api/v1/reconciliation/runs/{run_id}/items
```

## 56.5 Resolve Item

```http
POST /api/v1/reconciliation/items/{item_id}/resolve
```

---

# 57. Marketplace Publisher APIs

## 57.1 Register Publisher

```http
POST /api/v1/marketplace/publishers
```

## 57.2 List Publishers

```http
GET /api/v1/marketplace/publishers
```

## 57.3 Get Publisher

```http
GET /api/v1/marketplace/publishers/{publisher_id}
```

## 57.4 Update Publisher

```http
PATCH /api/v1/marketplace/publishers/{publisher_id}
```

## 57.5 Suspend Publisher

```http
POST /api/v1/marketplace/publishers/{publisher_id}/suspend
```

---

# 58. Marketplace App APIs

## 58.1 Create App

```http
POST /api/v1/marketplace/apps
```

## 58.2 List Apps

```http
GET /api/v1/marketplace/apps
```

## 58.3 Get App

```http
GET /api/v1/marketplace/apps/{app_id}
```

## 58.4 Update App

```http
PATCH /api/v1/marketplace/apps/{app_id}
```

## 58.5 Publish App

```http
POST /api/v1/marketplace/apps/{app_id}/publish
```

## 58.6 Unpublish App

```http
POST /api/v1/marketplace/apps/{app_id}/unpublish
```

---

# 59. Marketplace Version APIs

```http
POST /api/v1/marketplace/apps/{app_id}/versions
```

```http
GET /api/v1/marketplace/apps/{app_id}/versions
```

```http
GET /api/v1/marketplace/versions/{version_id}
```

```http
POST /api/v1/marketplace/versions/{version_id}/publish
```

```http
POST /api/v1/marketplace/versions/{version_id}/revoke
```

Artifact metadata should include:

```text
hash
signature
publisher
compatibility
permissions
artifact reference
```

---

# 60. Marketplace Purchase APIs

## 60.1 Purchase App

```http
POST /api/v1/marketplace/apps/{app_id}/purchase
```

## 60.2 List Purchases

```http
GET /api/v1/marketplace/purchases
```

## 60.3 Get Purchase

```http
GET /api/v1/marketplace/purchases/{purchase_id}
```

## 60.4 Cancel Purchase

```http
POST /api/v1/marketplace/purchases/{purchase_id}/cancel
```

---

# 61. Marketplace Installation APIs

## 61.1 Install

```http
POST /api/v1/marketplace/purchases/{purchase_id}/installations
```

## 61.2 List Installations

```http
GET /api/v1/marketplace/installations
```

## 61.3 Get Installation

```http
GET /api/v1/marketplace/installations/{installation_id}
```

## 61.4 Activate

```http
POST /api/v1/marketplace/installations/{installation_id}/activate
```

## 61.5 Disable

```http
POST /api/v1/marketplace/installations/{installation_id}/disable
```

## 61.6 Enable

```http
POST /api/v1/marketplace/installations/{installation_id}/enable
```

## 61.7 Update

```http
POST /api/v1/marketplace/installations/{installation_id}/update
```

## 61.8 Uninstall

```http
POST /api/v1/marketplace/installations/{installation_id}/uninstall
```

---

# 62. Marketplace Permissions APIs

```http
POST /api/v1/marketplace/apps/{app_id}/permissions
```

```http
GET /api/v1/marketplace/apps/{app_id}/permissions
```

```http
PATCH /api/v1/marketplace/permissions/{permission_id}
```

```http
DELETE /api/v1/marketplace/permissions/{permission_id}
```

Marketplace permissions should be reviewed before installation.

---

# 63. On-Premise Deployment APIs

## 63.1 Register Deployment

```http
POST /api/v1/licenses/deployments
```

## 63.2 List Deployments

```http
GET /api/v1/licenses/deployments
```

## 63.3 Get Deployment

```http
GET /api/v1/licenses/deployments/{deployment_id}
```

## 63.4 Update Deployment

```http
PATCH /api/v1/licenses/deployments/{deployment_id}
```

## 63.5 Revoke Deployment

```http
POST /api/v1/licenses/deployments/{deployment_id}/revoke
```

---

# 64. License Key APIs

## 64.1 Issue License

```http
POST /api/v1/licenses
```

## 64.2 List Licenses

```http
GET /api/v1/licenses
```

## 64.3 Get License

```http
GET /api/v1/licenses/{license_id}
```

## 64.4 Renew License

```http
POST /api/v1/licenses/{license_id}/renew
```

## 64.5 Revoke License

```http
POST /api/v1/licenses/{license_id}/revoke
```

## 64.6 Generate License Payload

```http
POST /api/v1/licenses/{license_id}/payload
```

The resulting payload must be cryptographically signed before deployment.

---

# 65. License Activation APIs

## 65.1 Activate

```http
POST /api/v1/licenses/{license_id}/activations
```

Request:

```json
{
  "deployment_id": "uuid",
  "hardware_fingerprint": "hashed-value"
}
```

## 65.2 List Activations

```http
GET /api/v1/licenses/{license_id}/activations
```

## 65.3 Get Activation

```http
GET /api/v1/license-activations/{activation_id}
```

## 65.4 Deactivate

```http
POST /api/v1/license-activations/{activation_id}/deactivate
```

---

# 66. Hardware Binding APIs

Hardware fingerprints should normally be hashed before persistence.

## 66.1 Bind Hardware

```http
POST /api/v1/deployments/{deployment_id}/hardware-bindings
```

## 66.2 List Bindings

```http
GET /api/v1/deployments/{deployment_id}/hardware-bindings
```

## 66.3 Remove Binding

```http
POST /api/v1/hardware-bindings/{binding_id}/remove
```

---

# 67. License Revocation APIs

## 67.1 Revoke

```http
POST /api/v1/licenses/{license_id}/revocations
```

## 67.2 Get Revocation

```http
GET /api/v1/license-revocations/{revocation_id}
```

## 67.3 Check Revocation

```http
POST /api/v1/licenses/check-revocation
```

Used by trusted deployment/runtime clients.

---

# 68. Webhook APIs

## 68.1 Register Endpoint

```http
POST /api/v1/webhooks/endpoints
```

## 68.2 List Endpoints

```http
GET /api/v1/webhooks/endpoints
```

## 68.3 Get Endpoint

```http
GET /api/v1/webhooks/endpoints/{endpoint_id}
```

## 68.4 Update Endpoint

```http
PATCH /api/v1/webhooks/endpoints/{endpoint_id}
```

## 68.5 Disable Endpoint

```http
POST /api/v1/webhooks/endpoints/{endpoint_id}/disable
```

## 68.6 Enable Endpoint

```http
POST /api/v1/webhooks/endpoints/{endpoint_id}/enable
```

## 68.7 Test Endpoint

```http
POST /api/v1/webhooks/endpoints/{endpoint_id}/test
```

---

# 69. Webhook Delivery APIs

## 69.1 List Deliveries

```http
GET /api/v1/webhooks/deliveries
```

## 69.2 Get Delivery

```http
GET /api/v1/webhooks/deliveries/{delivery_id}
```

## 69.3 Retry Delivery

```http
POST /api/v1/webhooks/deliveries/{delivery_id}/retry
```

---

# 70. Event APIs

Events are normally consumed through the message broker, but administrative inspection APIs are useful.

## 70.1 List Events

```http
GET /api/v1/events
```

Filters:

```text
event_type
tenant_id
correlation_id
occurred_after
occurred_before
```

## 70.2 Get Event

```http
GET /api/v1/events/{event_id}
```

## 70.3 Replay Event

```http
POST /api/v1/admin/events/{event_id}/replay
```

Replay must be restricted to authorized operators.

---

# 71. Outbox APIs

Administrative/operational APIs.

## 71.1 List Outbox Events

```http
GET /api/v1/admin/outbox/events
```

## 71.2 Get Outbox Event

```http
GET /api/v1/admin/outbox/events/{event_id}
```

## 71.3 Retry Outbox Event

```http
POST /api/v1/admin/outbox/events/{event_id}/retry
```

---

# 72. Inbox APIs

Administrative/operational APIs.

## 72.1 List Inbox Events

```http
GET /api/v1/admin/inbox/events
```

## 72.2 Get Inbox Event

```http
GET /api/v1/admin/inbox/events/{event_id}
```

---

# 73. Idempotency APIs

Idempotency keys are primarily an internal mechanism.

## 73.1 Get Idempotency Result

```http
GET /api/v1/idempotency/{key}
```

This endpoint should generally be restricted to the authenticated caller/context that created the key.

---

# 74. Audit APIs

## 74.1 List Audit Records

```http
GET /api/v1/audit
```

Filters:

```text
entity_type
entity_id
actor_id
action
occurred_after
occurred_before
```

## 74.2 Get Audit Record

```http
GET /api/v1/audit/{audit_id}
```

Audit records are append-only.

There should be no public:

```http
DELETE /audit/{id}
```

endpoint.

---

# 75. Configuration APIs

## 75.1 List Configuration

```http
GET /api/v1/configuration
```

## 75.2 Get Configuration

```http
GET /api/v1/configuration/{key}
```

## 75.3 Set Configuration

```http
PUT /api/v1/configuration/{key}
```

## 75.4 Delete Configuration

```http
DELETE /api/v1/configuration/{key}
```

Deletion should be implemented as a controlled configuration lifecycle where historical values must remain auditable.

---

# 76. Administrative Subscription APIs

## 76.1 Force Suspend

```http
POST /api/v1/admin/subscriptions/{subscription_id}/suspend
```

## 76.2 Force Reactivate

```http
POST /api/v1/admin/subscriptions/{subscription_id}/reactivate
```

## 76.3 Force Entitlement Rebuild

```http
POST /api/v1/admin/subscriptions/{subscription_id}/entitlement/rebuild
```

## 76.4 Recalculate Usage

```http
POST /api/v1/admin/subscriptions/{subscription_id}/usage/recalculate
```

Administrative operations must be audited.

---

# 77. Integration APIs — `p02_organization`

`p02_organization` remains the source of truth for tenant/company/master references.

Typical integration:

```text
p26_licensing → p02_organization
```

Example:

```http
GET /internal/v1/tenants/{tenant_id}
```

However, cross-platform calls should use the official `p02_organization` contract rather than hard-coded database access.

---

# 78. Integration APIs — `p01_identity`

Licensing should not manage:

```text
Passwords
Authentication sessions
Identity credentials
```

Identity integration should provide trusted claims such as:

```text
subject_id
tenant_id
roles
scopes
permissions
```

---

# 79. Integration APIs — `payment`

Recommended internal contract:

```http
POST /internal/v1/payment-requests
```

Payload:

```json
{
  "payment_request_id": "uuid",
  "customer_reference": "uuid",
  "billing_document_reference": "uuid",
  "amount": "299.00",
  "currency": "USD",
  "return_reference": "subscription_uuid"
}
```

Payment platform returns/events:

```text
PaymentInitiated
PaymentSucceeded
PaymentFailed
RefundCompleted
ChargebackCreated
```

Do not expose provider-specific credentials through licensing.

---

# 80. Integration APIs — `tax`

Recommended internal contract:

```http
POST /internal/v1/tax/calculate
```

Request:

```json
{
  "customer_reference": "uuid",
  "currency": "USD",
  "lines": [
    {
      "product_reference": "uuid",
      "quantity": "1",
      "unit_amount": "299.00"
    }
  ]
}
```

Response:

```json
{
  "tax_reference": "tax_123",
  "tax_amount": "53.82",
  "currency": "USD",
  "lines": []
}
```

The exact tax calculation rules belong to `tax`.

---

# 81. Integration Events — `accounting`

Licensing should publish commercial events.

Examples:

```text
InvoiceFinalized
CreditIssued
RefundCompleted
PaymentSucceeded
ChargebackCreated
SubscriptionActivated
SubscriptionCanceled
```

Accounting consumes these events and creates its own accounting records.

Licensing must never call:

```text
POST /journal-entries
```

inside its own domain.

---

# 82. Runtime Entitlement API

ERP gateway/runtime services should have a dedicated internal endpoint.

## 82.1 Resolve Entitlement

```http
GET /internal/v1/runtime/subscriptions/{subscription_id}/entitlement
```

## 82.2 Get Public Signing Keys

```http
GET /internal/v1/runtime/signing-keys
```

## 82.3 Validate Entitlement Version

```http
POST /internal/v1/runtime/entitlement/validate
```

Runtime consumers should prefer locally cached signed entitlement tokens instead of calling this endpoint on every request.

---

# 83. Service-to-Service Authentication

Internal APIs should require:

```text
mTLS
```

and/or:

```text
OAuth2 Client Credentials
```

Service identity should be explicit.

Example:

```text
payment
tax
accounting
p02_organization
```

Do not treat an arbitrary user JWT as sufficient authorization for internal privileged endpoints.

---

# 84. Required Scopes

Suggested scopes:

```text
licensing:read
licensing:write

catalog:read
catalog:write

subscription:read
subscription:write

entitlement:read
entitlement:write

usage:read
usage:write

billing:read
billing:write

marketplace:read
marketplace:write

license:read
license:write

admin:licensing
```

More restrictive scopes should be used for:

```text
signing key operations
license revocation
entitlement overrides
administrative subscription changes
event replay
outbox replay
```

---

# 85. Idempotency Requirements

Require `Idempotency-Key` for:

```text
POST /subscriptions
POST /subscriptions/{id}/activate
POST /subscriptions/{id}/change-plan
POST /subscriptions/{id}/pause
POST /subscriptions/{id}/resume
POST /subscriptions/{id}/cancel
POST /checkouts/{id}/confirm
POST /coupons/validate
POST /usage/events
POST /usage/events/batch
POST /billing/payment-requests
POST /billing/refunds
POST /marketplace/apps/{id}/purchase
POST /marketplace/purchases/{id}/installations
POST /licenses
POST /licenses/{id}/activations
```

The exact list can be extended whenever an operation is not naturally idempotent.

---

# 86. API Security Rules

Never accept authoritative tenant context from:

```http
X-Tenant-ID
```

Never accept:

```json
{
  "tenant_id": "another-tenant"
}
```

as authority when the authenticated context says otherwise.

Never expose:

```text
private signing keys
payment credentials
card data
secret webhook keys
database credentials
internal service credentials
```

---

# 87. API Versioning Rules

Current version:

```text
/api/v1
```

Breaking changes require:

```text
/api/v2
```

Non-breaking additions may remain within `v1`.

Do not silently change:

```text
enum meaning
field meaning
currency behavior
pricing semantics
subscription state semantics
event schema
```

---

# 88. Recommended API Lifecycle

Every command should follow:

```text
Authenticate
    ↓
Authorize
    ↓
Validate Input
    ↓
Load Aggregate
    ↓
Validate Business Invariants
    ↓
Execute Domain Command
    ↓
Persist State
    ↓
Write Outbox Event
    ↓
Commit
    ↓
Return Result
```

For asynchronous work:

```text
Command
  ↓
Persist
  ↓
Outbox
  ↓
202 Accepted
  ↓
Worker
  ↓
Domain Event
```

---

# 89. API-to-Event Mapping

| API Command | Event |
|---|---|
| Create Subscription | `SubscriptionCreated` |
| Activate Subscription | `SubscriptionActivated` |
| Change Plan | `SubscriptionUpdated` |
| Pause Subscription | `SubscriptionPaused` |
| Resume Subscription | `SubscriptionResumed` |
| Cancel Subscription | `SubscriptionCanceled` |
| Start Trial | `TrialStarted` |
| End Trial | `TrialEnded` |
| Publish Plan Version | `PlanVersionPublished` |
| Compile Entitlement | `EntitlementCompilationRequested` |
| Entitlement Updated | `EntitlementChanged` |
| Record Usage | `UsageRecorded` |
| Close Usage Period | `UsagePeriodClosed` |
| Detect Overage | `UsageOverageDetected` |
| Finalize Billing Document | `InvoiceFinalized` |
| Payment Request | `PaymentRequested` |
| Refund | `RefundRequested` |
| Marketplace Purchase | `MarketplacePurchaseCreated` |
| Install App | `MarketplaceInstallationActivated` |
| Issue License | `LicenseIssued` |
| Activate License | `LicenseActivated` |
| Revoke License | `LicenseRevoked` |

---

# 90. API Naming Rules

Prefer:

```text
POST /subscriptions/{id}/cancel
```

over:

```text
PATCH /subscriptions/{id}
{
  "status": "CANCELED"
}
```

Prefer domain commands for state transitions.

Prefer:

```text
POST /change-plan
```

for operations involving business logic.

Avoid RPC-style endpoints that bypass domain rules.

---

# 91. Delete Rules

Do not expose destructive deletes for historical records.

Avoid:

```text
DELETE /subscriptions/{id}
DELETE /usage/events/{id}
DELETE /audit/{id}
DELETE /invoices/{id}
DELETE /entitlements/{id}
DELETE /events/{id}
```

Use:

```text
cancel
archive
void
revoke
reverse
adjust
supersede
```

instead.

---

# 92. API Documentation

The FastAPI application should generate OpenAPI documentation:

```text
/openapi.json
/docs
/redoc
```

Production policy may disable public interactive documentation and expose it only internally.

---

# 93. OpenAPI Tags

Recommended tags:

```text
Health
Products
Product Versions
Categories
Features
Modules
Units
Currencies
Plans
Plan Versions
Prices
Add-ons
Subscriptions
Subscription Items
Subscription Changes
Schedules
Trials
Contracts
Checkout
Quotes
Discounts
Coupons
Entitlements
Entitlement Compilation
Signing Keys
Meters
Usage
Usage Periods
Usage Rating
Rate Limits
Billing
Proration
Dunning
Payments
Refunds
Chargebacks
Marketplace
Licenses
Deployments
Webhooks
Events
Audit
Configuration
Admin
```

---

# 94. Production Endpoint Summary

The platform should expose the following major endpoint groups:

```text
/api/v1/products/*
/api/v1/product-versions/*
/api/v1/product-categories/*
/api/v1/features/*
/api/v1/modules/*
/api/v1/units/*
/api/v1/currencies/*

/api/v1/plans/*
/api/v1/plan-versions/*
/api/v1/prices/*
/api/v1/addons/*
/api/v1/addon-versions/*

/api/v1/subscriptions/*
/api/v1/subscription-items/*
/api/v1/subscription-changes/*
/api/v1/subscription-schedules/*
/api/v1/trial-policies/*
/api/v1/contracts/*

/api/v1/checkouts/*
/api/v1/quotes/*
/api/v1/discounts/*
/api/v1/coupons/*

/api/v1/entitlements/*
/api/v1/entitlement-compilations/*
/api/v1/entitlement-overrides/*

/api/v1/meters/*
/api/v1/usage/*
/api/v1/usage-periods/*
/api/v1/rate-limit-policies/*

/api/v1/billing/*
/api/v1/proration/*
/api/v1/dunning/*
/api/v1/reconciliation/*

/api/v1/marketplace/*
/api/v1/licenses/*

/api/v1/webhooks/*
/api/v1/events/*
/api/v1/audit/*
/api/v1/configuration/*

/api/v1/admin/*
/api/v1/internal/*
```

---

# 95. Final Boundary

The API surface must preserve the platform boundary:

```text
p26_licensing
│
├── Catalog
├── Plans
├── Pricing
├── Add-ons
├── Subscription
├── Trials
├── Contracts
├── Checkout
├── Quotes
├── Discounts
├── Entitlement
├── Metering
├── Usage
├── Overage
├── Commercial Billing Coordination
├── Marketplace
├── Licensing
├── Webhooks
├── Events
└── Audit
```

External platform responsibilities remain:

```text
p02_organization
    → Tenant / Company / Branch master data

p01_identity
    → Authentication / IAM / identity

payment
    → Payment provider execution

tax
    → Tax calculation

accounting
    → GL / AR / AP / revenue / financial accounting
```

The licensing API must connect these platforms through **versioned APIs and domain events**, never by sharing databases or mixing domain logic.
