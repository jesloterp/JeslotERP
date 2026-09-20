# [ARCHIVED encyclopedia] p26_licensing — Developer Guide

> **HYG-015:** Ship architecture is [`V2_LICENSING_GUIDE.md`](V2_LICENSING_GUIDE.md). This file is retained as the deep v1 encyclopedia only. Do not mount its `/api/v1/products` paths.

# p26_licensing — Developer Guide

**Version:** 1.2  
**Last reviewed:** 2026-09-08  
**Platform:** JeslotERP  
**Service / package:** `platforms.p26_licensing`  
**Primary Stack:** Python 3.12+, FastAPI, SQLAlchemy 2.x, PostgreSQL, Redis  
**Architecture:** Modular Monolith / Microservice-ready, DDD, Hexagonal Architecture, CQRS where useful, Transactional Outbox, Inbox/Idempotency, Event-Driven Processing

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial developer guide baseline. |
| 1.1 | 2026-09-08 | Optimistic concurrency standardized on the integer `version` counter (§24), matching the production schema (`version_id_col`) and the ORG/Configuration platforms; added a schema-alignment note reconciling the seven guide-referenced tables not yet in the schema (Appendix A); added an enterprise Data Retention, Privacy & Compliance section (§60.1); referenced the schema's money `Numeric` precision rule (§8). |
| 1.2 | 2026-09-08 | Renumbered platform to `p26_licensing` (docs `04_licensing`). Mapped existing deps to `p01_identity` / `p02_organization`. Removed invented numbers for future platforms (`payment`, `tax`, `accounting`). Added repository integration contract; see `docs/PLATFORM_REGISTRY.md`. |

---

# 0. Repository Integration Contract

| Item | Frozen value |
|---|---|
| Docs folder | `docs/platforms/26_licensing/` |
| Platform package | `platforms.p26_licensing` |
| `ModulePlugin.name` | `p26_licensing` |
| Module dependencies | `["p01_identity", "p02_organization"]` (+ logical `payment` / `tax` / `accounting` when built) |
| PostgreSQL schema | `licensing` |
| Public API | `/api/v1` (licensing host) / mounted routes under app |
| Numbering authority | [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md) |

**Assigned platforms only:** `p01_identity`, `p02_organization`, `p03_configuration`, `p26_licensing`.

**Do not invent numbers** for future platforms in this guide. Spell them as logical names: `payment`, `tax`, `accounting`, `business_partner`, `notification`.

---

## 1. Purpose

`p26_licensing` is the dedicated **Subscription & Licensing Platform**.

Its responsibility is to answer:

> What commercial product did a tenant purchase, what subscription is active, and what capabilities and usage limits is the tenant entitled to right now?

The platform is intentionally separated from:

- Identity / IAM
- Payment provider processing
- Tax engine
- Accounting / Finance / General Ledger
- ERP business modules
- Notification delivery infrastructure

The licensing platform owns commercial subscription state and runtime entitlement state, but it is **not an accounting system**.

---

## 2. Architectural Boundary

### 2.1 What `p26_licensing` owns

`p26_licensing` owns:

- Product catalog
- Product versions
- Features
- Plans
- Plan versions
- Prices and pricing models
- Add-ons
- Subscriptions
- Subscription items
- Subscription changes
- Subscription schedules
- Trials
- Contracts / commercial commitments
- Checkout
- Quotes
- Discounts
- Coupons
- Entitlements
- Entitlement compilation
- Signed entitlement tokens
- Meter definitions
- Usage events
- Usage aggregation
- Usage rating
- Quotas
- Rate limits
- Overage calculations
- Commercial billing-cycle coordination
- Commercial invoice documents, where required
- Proration calculations
- Dunning state coordination
- Marketplace subscription/install state
- On-premise license issuance and activation
- Webhooks
- Domain events
- Outbox / inbox
- Idempotency
- Audit history
- Licensing configuration

### 2.2 What it does NOT own

Do **not** put these domains inside `p26_licensing`:

```text
General Ledger
Journal Entries
Chart of Accounts
Accounts Receivable Accounting
Accounts Payable
Bank Reconciliation
Financial Statements
Revenue Recognition
Deferred Revenue Accounting
Tax Ledger
Payment Gateway Processing
Card Vault
Bank Account Processing
Identity Authentication
User Passwords
ERP Business Transactions
```

These belong to dedicated platforms.

---

# 3. Platform Ecosystem

Recommended platform separation:

```text
                         ┌──────────────────┐
                         │     p02_organization      │
                         │ Tenant / Company  │
                         │ Branch / Master   │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │  p01_identity    │
                         │ IAM / Auth / RBAC │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │  p26_licensing   │
                         │ Subscription      │
                         │ Entitlement       │
                         │ Metering          │
                         └─────┬──────┬──────┘
                               │      │
                 ┌─────────────┘      └─────────────┐
                 ▼                                  ▼
        ┌──────────────────┐               ┌──────────────────┐
        │   payment    │               │     tax      │
        │ Provider Adapter │               │ Tax Calculation  │
        └────────┬─────────┘               └────────┬─────────┘
                 │                                  │
                 └────────────────┬─────────────────┘
                                  ▼
                         ┌──────────────────┐
                         │ accounting   │
                         │ GL / AR / Revenue │
                         └──────────────────┘
```

### Important rule

Never create direct database joins across platforms.

Use:

- REST/gRPC APIs for synchronous operations
- Events for asynchronous integration
- Stable external IDs
- Idempotency keys
- Versioned contracts

---

# 4. Core Design Principles

## 4.1 Commercial Definition

Defines:

> What can be sold and for how much?

Includes:

```text
Product
  ↓
Product Version
  ↓
Feature
  ↓
Plan
  ↓
Plan Version
  ↓
Price
```

## 4.2 Commercial Acquisition

Defines:

> Who purchased what?

Includes:

```text
Checkout
  ↓
Subscription
  ↓
Subscription Items
  ↓
Add-ons
  ↓
Contract
```

## 4.3 Runtime Enforcement

Defines:

> What can the tenant do right now?

Includes:

```text
Subscription
  ↓
Entitlement Compilation
  ↓
Materialized Entitlement
  ↓
Signed JWT
  ↓
API Gateway
  ↓
Quota / Rate Limit / Feature Enforcement
```

---

# 5. Project Structure

Recommended structure:

```text
app/
├── main.py
│
├── core/
│   ├── config.py
│   ├── database.py
│   ├── security.py
│   ├── exceptions.py
│   ├── enums.py
│   └── logging.py
│
├── shared/
│   ├── base_models.py
│   ├── money.py
│   ├── pagination.py
│   ├── types.py
│   ├── events.py
│   └── errors.py
│
├── modules/
│   ├── catalog/
│   ├── plans/
│   ├── pricing/
│   ├── addons/
│   ├── subscriptions/
│   ├── contracts/
│   ├── trials/
│   ├── checkout/
│   ├── quotes/
│   ├── discounts/
│   ├── coupons/
│   ├── entitlement/
│   ├── metering/
│   ├── usage/
│   ├── rating/
│   ├── billing/
│   ├── marketplace/
│   ├── licensing/
│   ├── webhooks/
│   ├── events/
│   ├── audit/
│   └── configuration/
│
├── infrastructure/
│   ├── postgres/
│   ├── redis/
│   ├── messaging/
│   ├── kms/
│   └── external/
│
└── workers/
    ├── entitlement_worker.py
    ├── usage_worker.py
    ├── billing_worker.py
    ├── dunning_worker.py
    ├── webhook_worker.py
    └── reconciliation_worker.py
```

Each module should normally contain:

```text
module/
├── domain/
├── application/
├── infrastructure/
├── api/
└── schemas/
```

---

# 6. Database Schema Domains

## 6.1 Catalog

Main tables:

```text
license_product
license_product_category
license_product_version
license_feature
license_feature_dependency
license_module
license_module_feature
license_unit
license_currency
```

### Product

Represents a sellable product.

Examples:

```text
NEXTERP
HR
CRM
Inventory
API Platform
Marketplace App
Storage
```

### Product Version

Product versions provide temporal stability.

A published product version should not be mutated in a way that changes historical meaning.

---

# 7. Plans and Plan Versions

Main tables:

```text
license_plan
license_plan_version
license_plan_version_feature
```

A subscription should reference the **plan version**, not only the plan.

Example:

```text
Plan:
    Enterprise

Plan Version:
    Enterprise v1
    Enterprise v2
    Enterprise v3
```

This guarantees that changing today's Enterprise pricing does not silently change an existing customer's commercial agreement.

## Publication rule

Before publication:

```text
DRAFT
```

After publication:

```text
PUBLISHED
```

After retirement:

```text
RETIRED
```

Published versions should be treated as immutable.

---

# 8. Pricing Engine

Supported pricing models:

```text
FREE
FLAT
PER_UNIT
TIERED
VOLUME
GRADUATED
PACKAGE
STAIR_STEP
METERED
HYBRID
```

Main tables:

```text
license_price
license_price_tier
license_price_component
```

## Money rules

Never use:

```python
float
```

for monetary calculations.

Use:

```python
Decimal
```

or integer minor units.

Example:

```text
USD 100.50
```

should be represented safely as:

```text
Decimal("100.50")
```

At the database layer, every monetary/quantity column must be `Numeric` with an explicit precision/scale (e.g. `Numeric(18, 6)`), never unbounded — see the production schema's normative money rule (Schema §64.1). Choose scale ≥ the currency's `minor_unit`; rating and proration may need scale 6+.

Currency configuration must support:

- ISO 4217 code
- Numeric code
- Minor units
- Symbol
- Cash rounding increment
- Rounding mode

---

# 9. Add-ons

Main tables:

```text
license_addon
license_addon_version
license_addon_feature
```

An add-on modifies or extends a subscription.

Examples:

```text
+10 Users
+100 GB Storage
Advanced Analytics
Payroll
Extra API Capacity
```

Add-ons should also follow temporal versioning.

---

# 10. Subscription Engine

Main tables:

```text
license_subscription
license_subscription_item
license_subscription_change
license_subscription_schedule
license_subscription_schedule_phase
license_subscription_trial
license_trial_policy
license_contract
license_contract_line
license_contract_entitlement
license_subscription_pause
license_cancellation
license_subscription_snapshot
```

## Subscription lifecycle

Recommended states:

```text
DRAFT
TRIALING
ACTIVE
PAST_DUE
PAUSED
SUSPENDED
CANCELED
EXPIRED
```

Typical lifecycle:

```text
DRAFT
  ↓
TRIALING
  ↓
ACTIVE
  ↓
PAST_DUE
  ↓
SUSPENDED
  ↓
CANCELED
```

Not every subscription must use every state.

---

# 11. Subscription Changes

Changes include:

```text
UPGRADE
DOWNGRADE
QUANTITY_INCREASE
QUANTITY_DECREASE
ADDON_ADD
ADDON_REMOVE
PRICE_CHANGE
CURRENCY_CHANGE
INTERVAL_CHANGE
PAUSE
RESUME
CANCEL
REACTIVATE
```

Effective strategies:

```text
IMMEDIATE
END_OF_TERM
SCHEDULED
```

Every change should retain:

- Previous state
- Requested state
- Effective date
- Actor
- Reason
- Proration result
- Correlation ID

---

# 12. Subscription Schedules

Subscription schedules support phased commercial agreements.

Example:

```text
Phase 1
Jan-Mar
Starter
$50

Phase 2
Apr-Jun
Professional
$100

Phase 3
Jul-Dec
Enterprise
$300
```

Never overwrite historical phases.

Create a new phase or schedule revision.

---

# 13. Trials

Trial functionality should support:

- Trial duration
- Trial start
- Trial end
- Payment method required
- Auto-conversion
- Trial cancellation
- Trial extension
- Trial policy
- Trial eligibility

Example:

```text
Subscription
    ↓
TRIALING
    ↓
Trial expires
    ↓
ACTIVE
```

or:

```text
TRIALING
    ↓
Trial expires
    ↓
CANCELED
```

---

# 14. Contracts

Enterprise customers may have contractual commitments.

Supported concepts:

```text
Contract
Contract Lines
Minimum Commitment
Committed Quantity
Committed Amount
Renewal Date
Notice Period
Contract Entitlements
```

Contract entitlements should participate in entitlement compilation.

---

# 15. Entitlement Engine

The entitlement engine is the most performance-sensitive part of the platform.

The original architecture intentionally avoids evaluating:

```text
Tenant
 → Subscription
 → Plan
 → Plan Version
 → Feature
 → Module
 → Add-on
```

on every ERP request.

Instead:

```text
Commercial State
      ↓
Entitlement Compiler
      ↓
Materialized Entitlement
      ↓
Signed Token
      ↓
Gateway
```

---

# 16. Entitlement Tables

Main tables:

```text
license_entitlement
license_entitlement_feature
license_entitlement_limit
license_entitlement_override
license_entitlement_source
license_entitlement_compilation
license_entitlement_token
license_signing_key
```

## Precedence

Recommended precedence:

```text
1. Base Plan
2. Plan Version
3. Add-on
4. Contract
5. Promotional Grant
6. Administrative Override
7. Final Effective Entitlement
```

The exact precedence must be deterministic and versioned.

---

# 17. Entitlement Operations

Supported operations:

```text
SET
ADD
SUBTRACT
MULTIPLY
ENABLE
DISABLE
```

Example:

```text
Plan:
    max_users = 50

Add-on:
    +10 users

Contract:
    +20 users

Final:
    max_users = 80
```

Every final value should preserve its source information.

---

# 18. Entitlement Compilation

Trigger compilation whenever relevant commercial state changes.

Examples:

```text
SubscriptionCreated
SubscriptionUpdated
SubscriptionCanceled
SubscriptionPaused
SubscriptionResumed
PlanChanged
AddOnAdded
AddOnRemoved
ContractChanged
AdministrativeOverrideChanged
PromotionApplied
```

Flow:

```text
Transaction
    ↓
Transactional Outbox
    ↓
SubscriptionChanged
    ↓
Entitlement Worker
    ↓
Load Plan Version
    ↓
Load Features
    ↓
Load Add-ons
    ↓
Load Contract
    ↓
Load Promotions
    ↓
Load Overrides
    ↓
Resolve Precedence
    ↓
Generate Final Entitlement
    ↓
Calculate Payload Hash
    ↓
Sign Token
    ↓
Persist
    ↓
Publish EntitlementChanged
```

---

# 19. Signed Entitlement JWT

The token should contain only what the runtime needs.

Example conceptual payload:

```json
{
  "iss": "p26_licensing",
  "tenant_id": "...",
  "subscription_id": "...",
  "entitlement_version": 42,
  "features": [
    "FINANCE",
    "HR",
    "INVENTORY"
  ],
  "limits": {
    "max_users": 50,
    "api_requests_month": 100000
  },
  "iat": 1790000000,
  "exp": 1790003600,
  "jti": "..."
}
```

Use asymmetric signing such as RSA/ECDSA.

Private keys must not be hard-coded or stored as ordinary database secrets.

Use:

```text
KMS
HSM
Secret Manager
```

Support:

```text
kid
key rotation
key activation
key retirement
revocation
```

---

# 20. Runtime Enforcement

Request path:

```text
HTTP Request
    ↓
API Gateway
    ↓
Verify Entitlement
    ↓
Feature Check
    ↓
Rate Limit Check
    ↓
Quota Check
    ↓
ERP Business Logic
    ↓
Usage Recording
```

Feature unavailable:

```text
403 Forbidden
```

Rate limit exceeded:

```text
429 Too Many Requests
```

Hard quota exceeded:

```text
422 / domain-specific quota error
```

The final HTTP mapping should be standardized across the platform.

---

# 21. Metering and Usage

Main tables:

```text
license_meter
license_meter_event
license_usage_period
license_usage
license_usage_ledger
license_usage_reset
license_usage_overage
license_usage_rating
```

Supported aggregation:

```text
COUNT
SUM
MAX
MIN
UNIQUE
LAST
AVERAGE
```

Examples:

```text
API requests
Storage GB
Messages sent
Employees
Invoices created
Documents processed
AI tokens
Transactions
```

---

# 22. Usage Ledger

The usage ledger is an **immutable metering ledger**.

It is not financial accounting.

Entry types:

```text
CONSUME
RESERVE
RELEASE
ADJUSTMENT
RESET
REVERSAL
```

Rules:

- Append-only
- Idempotent
- Timestamped
- Tenant scoped
- Subscription scoped
- Meter scoped
- Correlated to source event
- Never silently deleted

If a usage event is wrong, create a reversal or adjustment event.

---

# 23. Idempotent Usage Ingestion

Every usage event should have an idempotency key.

Example:

```json
{
  "event_id": "evt_123",
  "idempotency_key": "invoice-created-abc-001",
  "tenant_id": "tenant_123",
  "subscription_id": "sub_123",
  "meter_id": "invoice_created",
  "quantity": 1,
  "occurred_at": "2026-09-08T10:00:00Z"
}
```

Duplicate events must not double-charge or double-consume quota.

---

# 24. Quota Concurrency

For hard limits, race conditions must be prevented.

Example:

```text
Maximum companies = 3
Current companies = 2
Two requests arrive simultaneously
```

Both requests must not create company #3 and company #4.

Use optimistic locking on the base-class integer `version` counter (the schema's `version_id_col`, consistent with the ORG/Configuration platforms — there is no separate `row_version`):

```sql
UPDATE license_usage
SET current_value = current_value + 1,
    version = version + 1
WHERE id = :usage_id
  AND version = :expected_version;
```

If zero rows are updated:

```text
Retry / abort
```

For very high-throughput counters, prefer an atomic conditional increment
(`... SET current_value = current_value + 1 WHERE id = :id AND current_value + 1 <= limit_value RETURNING current_value`) or Redis-backed reservation, falling back to the `version` guard above for durable correctness.

---

# 25. Redis Rate Limiting

Redis is suitable for high-throughput runtime rate limiting.

Supported algorithms:

```text
TOKEN_BUCKET
SLIDING_WINDOW
FIXED_WINDOW
LEAKY_BUCKET
```

Recommended use:

```text
Redis
    → fast runtime rate limiting

PostgreSQL
    → durable authoritative usage
```

Redis should not become the only durable source of commercial usage.

---

# 26. Usage Rating

Metered usage can be transformed into commercial charges.

Flow:

```text
Usage Event
    ↓
Aggregation
    ↓
Usage Period
    ↓
Price Lookup
    ↓
Tier Calculation
    ↓
Discount
    ↓
Tax Request
    ↓
Rated Amount
```

Example:

```text
Included API calls: 100,000
Actual calls:       125,000
Overage:             25,000
Price:                $0.001/call
Charge:               $25
```

The rated result should be immutable after commercial finalization.

---

# 27. Checkout

Main tables:

```text
license_checkout
license_checkout_item
```

Checkout should be treated as a temporary commercial calculation.

Typical flow:

```text
Select Product
    ↓
Select Plan
    ↓
Select Currency
    ↓
Select Add-ons
    ↓
Apply Coupon
    ↓
Calculate Proration
    ↓
Calculate Tax
    ↓
Create Payment Request
    ↓
Payment Platform
    ↓
Subscription Activation
```

Do not store payment card details in `p26_licensing`.

---

# 28. Quotes

Main tables:

```text
license_quote
license_quote_line
license_quote_conversion
```

Quotes should support:

- Expiration
- Customer
- Currency
- Plan
- Add-ons
- Discounts
- Tax result reference
- Commercial totals
- Version
- Acceptance
- Conversion to subscription

Accepted quotes should preserve the commercial snapshot used for conversion.

---

# 29. Discounts and Coupons

Main tables:

```text
license_discount
license_discount_rule
license_coupon
license_coupon_condition
license_coupon_usage
```

Coupon validation can include:

```text
Expiration
Maximum Redemptions
Per-Customer Limit
Plan Eligibility
Product Eligibility
New Customer
Minimum Quantity
Minimum Amount
Country / Market
Billing Interval
```

Validation must be deterministic.

---

# 30. Proration

Immediate upgrades may require proration.

Example:

```text
Current plan:
$100/month

New plan:
$300/month

50% period remaining
```

Unused old plan credit:

```text
$50
```

Remaining new plan charge:

```text
$150
```

Net:

```text
$100
```

Downgrades may be scheduled for end-of-term depending on product policy.

Proration calculation must store:

- Old price
- New price
- Period start
- Period end
- Effective timestamp
- Used duration
- Remaining duration
- Credit
- Charge
- Net amount
- Currency
- Calculation version

---

# 31. Commercial Billing Boundary

`p26_licensing` can coordinate subscription billing state and commercial documents.

Possible tables:

```text
license_billing_account
license_billing_cycle
license_invoice
license_invoice_line
license_proration
license_customer_balance
license_balance_transaction
license_dunning_policy
license_dunning_case
license_payment_reference
license_refund_reference
license_chargeback_reference
license_external_reference
license_reconciliation_run
license_reconciliation_item
```

### Critical boundary

These records do not turn `p26_licensing` into the accounting system.

Accounting remains:

```text
accounting
```

Payment processing remains:

```text
payment
```

Tax calculation remains:

```text
tax
```

---

# 32. Payment Platform Integration

Licensing sends a commercial payment request.

Example:

```text
p26_licensing
    ↓
PaymentRequested
    ↓
payment
    ↓
Payment Provider
```

Possible providers:

```text
Stripe
Adyen
Razorpay
PayPal
Bank Transfer Adapter
Other regional providers
```

`p26_licensing` should not contain provider-specific card-processing logic.

Store provider references only:

```text
payment_intent_id
payment_transaction_id
provider
provider_reference
payment_status
```

---

# 33. Tax Platform Integration

Tax calculation should be delegated to:

```text
tax
```

Licensing provides the commercial context:

```text
Customer
Ship-to / Bill-to reference
Product
Plan
Price
Quantity
Currency
Discount
Tax category
```

Tax platform returns:

```text
Taxable Amount
Tax Lines
Tax Rate
Tax Amount
Jurisdiction
Tax Reference
Calculation Version
```

Persist the resulting tax snapshot/reference required for the commercial document.

---

# 34. Accounting Integration

Accounting is downstream.

Example event:

```text
InvoiceFinalized
```

can be consumed by:

```text
accounting
```

Possible accounting events:

```text
InvoiceFinalized
CreditIssued
RefundCompleted
PaymentSucceeded
PaymentFailed
ChargebackCreated
SubscriptionActivated
SubscriptionCanceled
```

Accounting decides how those events affect:

```text
AR
Revenue
Deferred Revenue
Receivables
Cash
Tax Payable
```

Licensing must not create GL journal entries.

---

# 35. Dunning

Typical commercial lifecycle:

```text
OPEN
  ↓
PAYMENT_ATTEMPT
  ↓
PAID
```

Failure:

```text
PAYMENT_ATTEMPT
  ↓
PAST_DUE
  ↓
GRACE_PERIOD
  ↓
SUSPENDED
  ↓
CANCELED
```

Grace periods and retry schedules should be policy-driven rather than hard-coded.

Example policy:

```text
Day 1  → warning
Day 3  → warning
Day 7  → final warning
Day 7+ → suspend
Day 30 → cancel
```

---

# 36. Marketplace

Main tables:

```text
license_marketplace_publisher
license_marketplace_app
license_marketplace_version
license_marketplace_purchase
license_marketplace_installation
license_marketplace_permission
```

Lifecycle:

```text
Browse
  ↓
Purchase
  ↓
Purchase Record
  ↓
Install
  ↓
Fetch Artifact
  ↓
Verify Artifact
  ↓
Deploy
  ↓
ACTIVE
  ↓
Update / Disable / Uninstall
```

Artifact metadata should include:

```text
Hash
Signature
Publisher
Version
Compatibility
Permissions
Artifact URL
Revocation State
```

Never blindly execute an unsigned marketplace artifact.

---

# 37. On-Premise Licensing

For on-premise deployments, the customer may control the database.

Therefore database values cannot be treated as the final trust boundary.

Use:

```text
Master Licensing Server
       ↓
License Payload
       ↓
RSA/ECDSA Signature
       ↓
Customer Deployment
       ↓
Local Verification
```

Example payload:

```json
{
  "tenant": "tenant-x",
  "max_users": 50,
  "modules": [
    "FIN",
    "HR"
  ],
  "expires_at": "2027-01-01T00:00:00Z"
}
```

The server validates:

```text
Signature
Hardware Binding
Expiration
Deployment ID
Revocation State
```

Invalid signature or hardware mismatch should put the deployment into a controlled locked state.

---

# 38. Signing Key Rotation

Never use one permanent key forever.

Recommended lifecycle:

```text
GENERATED
  ↓
ACTIVE
  ↓
ROTATING
  ↓
RETIRED
  ↓
REVOKED
```

Tokens should contain:

```text
kid
```

Consumers retrieve the correct public key by key ID.

---

# 39. Transactional Outbox

Every important state change should use an outbox.

Example:

```text
BEGIN TRANSACTION

UPDATE subscription

INSERT outbox_event

COMMIT
```

Only after the transaction commits does the event publisher deliver:

```text
SubscriptionUpdated
```

This prevents:

```text
DB updated
but event lost
```

---

# 40. Inbox Pattern

Consumers should record processed events.

Example:

```text
license_inbox_event
```

Processing:

```text
Receive Event
    ↓
Check Event ID
    ↓
Already processed?
   / \
 Yes  No
 ↓     ↓
Skip  Process
       ↓
    Mark Processed
```

This makes event handling idempotent.

---

# 41. Idempotency API

Use:

```text
license_idempotency_key
```

for commands such as:

```text
Create Subscription
Change Plan
Cancel Subscription
Create Checkout
Apply Coupon
Record Usage
Create Marketplace Purchase
Activate License
```

Client supplies:

```http
Idempotency-Key: <unique-key>
```

Same request repeated with the same key should return the original result where applicable.

---

# 42. Domain Events

Recommended events:

```text
ProductCreated
ProductVersionPublished

PlanCreated
PlanVersionPublished

SubscriptionCreated
SubscriptionActivated
SubscriptionUpdated
SubscriptionPaused
SubscriptionResumed
SubscriptionCanceled
SubscriptionExpired

SubscriptionItemAdded
SubscriptionItemRemoved

TrialStarted
TrialEnded

EntitlementCompilationRequested
EntitlementChanged

UsageRecorded
UsagePeriodClosed
UsageLimitReached
UsageOverageDetected

QuoteCreated
QuoteAccepted
QuoteExpired

CouponApplied
CouponRejected

InvoiceDrafted
InvoiceFinalized
InvoiceVoided

PaymentRequested
PaymentSucceeded
PaymentFailed

RefundRequested
RefundCompleted
ChargebackCreated

MarketplacePurchaseCreated
MarketplaceInstallationActivated
MarketplaceInstallationDisabled
MarketplaceInstallationUninstalled

LicenseIssued
LicenseActivated
LicenseRevoked
```

Events should be versioned.

Example:

```text
subscription.activated.v1
subscription.activated.v2
```

---

# 43. Event Envelope

Recommended event structure:

```json
{
  "event_id": "evt_123",
  "event_type": "subscription.activated",
  "event_version": 1,
  "occurred_at": "2026-09-08T10:00:00Z",
  "producer": "p26_licensing",
  "tenant_id": "tenant_123",
  "correlation_id": "corr_123",
  "causation_id": "cmd_123",
  "data": {}
}
```

Do not expose internal database structure as the public event contract.

---

# 44. API Design

Recommended API grouping:

```text
/api/v1/catalog
/api/v1/products
/api/v1/plans
/api/v1/prices
/api/v1/addons
/api/v1/subscriptions
/api/v1/contracts
/api/v1/trials
/api/v1/checkout
/api/v1/quotes
/api/v1/coupons
/api/v1/discounts
/api/v1/entitlements
/api/v1/meters
/api/v1/usage
/api/v1/billing
/api/v1/marketplace
/api/v1/licenses
/api/v1/webhooks
```

Use:

```text
GET
POST
PATCH
DELETE
```

carefully according to resource lifecycle.

Published commercial records should generally not support destructive updates.

---

# 45. Authentication and Authorization

Authentication belongs to:

```text
p01_identity
```

Licensing should receive a trusted identity context.

Example:

```text
JWT
 ↓
Identity Gateway
 ↓
Licensing API
```

Licensing should authorize based on:

```text
tenant_id
subject_id
roles
scopes
permissions
platform claims
```

Never trust a client-supplied:

```http
X-Tenant-ID
```

as the authoritative tenant identity.

Tenant identity must come from the verified security context.

---

# 46. Tenant Isolation

Every tenant-owned entity should contain:

```text
tenant_id
```

Where appropriate:

```text
company_id
branch_id
```

Do not blindly add tenant columns to global catalog entities.

Example:

```text
Global:
license_currency
license_product
license_feature
```

Tenant scoped:

```text
license_subscription
license_usage
license_entitlement
license_checkout
```

---

# 47. Cross-Platform References

Do not create foreign keys across service databases.

Instead:

```text
tenant_id
customer_id
company_id
branch_id
external_reference_id
```

are logical references.

Example:

```text
p26_licensing.subscription.customer_id
```

may refer to an entity managed by:

```text
p02_organization
```

but there should be no SQL foreign key to another database.

---

# 48. Audit

Main table:

```text
license_audit
```

Audit should capture:

```text
Actor
Tenant
Action
Entity Type
Entity ID
Before State
After State
Timestamp
IP / Request Context where appropriate
Correlation ID
Reason
```

Audit records should be append-only.

Never update audit history to hide previous actions.

---

# 49. Immutable Data

Treat these as immutable after finalization:

```text
Published Plan Versions
Published Price Versions
Finalized Commercial Calculations
Usage Ledger Entries
Audit Records
Processed Domain Events
Subscription Snapshots
Issued License Records
Finalized Invoice Snapshots
```

Correction should normally happen through:

```text
new version
reversal
adjustment
superseding record
```

not destructive mutation.

---

# 50. Database Transactions

Use database transactions around business invariants.

Example:

```text
BEGIN

Validate subscription
Validate plan
Validate quantity
Calculate proration
Update subscription
Create subscription change
Create entitlement compilation request
Create outbox event

COMMIT
```

Do not publish the event before the transaction commits.

---

# 51. Background Workers

Recommended workers:

### Entitlement Worker

```text
Subscription Event
 → Compile Entitlement
 → Sign Token
 → Publish EntitlementChanged
```

### Usage Worker

```text
Usage Events
 → Aggregate
 → Update Usage Period
 → Detect Overage
```

### Billing Worker

```text
Billing Cycle
 → Gather Usage
 → Rate Usage
 → Calculate Commercial Charges
 → Create Billing Document
```

### Dunning Worker

```text
Past Due
 → Retry Policy
 → Notification Event
 → Suspension
```

### Webhook Worker

```text
Outbox Event
 → Webhook Delivery
 → Retry
 → Dead Letter
```

---

# 52. Retry Strategy

External and asynchronous operations should use:

```text
Exponential Backoff
Jitter
Maximum Retry Count
Dead Letter Queue
Idempotency
```

Example:

```text
1m
5m
15m
1h
6h
24h
```

Exact policy should be configurable.

Never retry permanently invalid requests indefinitely.

---

# 53. Webhooks

Main tables:

```text
license_webhook_endpoint
license_webhook_delivery
```

Webhook delivery should include:

```text
event_id
endpoint_id
attempt_count
status
response_code
next_attempt_at
delivered_at
last_error
signature
```

Sign outgoing webhooks.

Consumers must verify signatures.

---

# 54. Observability

Every request should have:

```text
request_id
correlation_id
trace_id
tenant_id
```

Metrics should include:

```text
subscription_creation_total
subscription_change_total
entitlement_compilation_duration
entitlement_compilation_failure_total
usage_ingestion_total
usage_ingestion_failure_total
quota_rejection_total
rate_limit_rejection_total
billing_document_total
payment_event_total
webhook_delivery_failure_total
event_processing_lag
```

Do not log secrets, payment credentials, private keys, or sensitive tokens.

---

# 55. Caching

Good cache candidates:

```text
Published plan versions
Published prices
Currency metadata
Feature definitions
Public signing keys
Runtime entitlement tokens
```

Do not cache mutable authorization decisions indefinitely.

Use explicit TTL and invalidation through:

```text
EntitlementChanged
PlanVersionPublished
PriceChanged
```

---

# 56. Performance Strategy

The critical runtime path should avoid:

```text
5+ relational joins
```

Use:

```text
Materialized Entitlements
Redis
Signed Tokens
Read Models
```

The target architecture is:

```text
Write complexity
      ↓
Compile once
      ↓
Read millions of times
```

---

# 57. Database Indexing

Important indexes include:

```text
tenant_id
subscription_id
customer_id
status
effective_at
starts_at
ends_at
meter_id
usage_period_id
idempotency_key
event_id
correlation_id
external_reference
```

Composite indexes should follow actual query patterns.

Example:

```text
(tenant_id, status)
(tenant_id, subscription_id)
(subscription_id, effective_at)
(tenant_id, meter_id, period_start)
```

---

# 58. High-Volume Usage Tables

Usage may become extremely large.

Plan for:

```text
Partitioning
Archival
Retention Policies
Time-based partitions
Tenant-aware indexes
Compression where supported
Batch ingestion
```

Typical partition candidates:

```text
license_meter_event
license_usage_ledger
license_audit
license_webhook_delivery
```

Do not introduce partitioning blindly on day one; design schemas so it can be introduced without changing the domain model.

---

# 59. Configuration

Main table:

```text
license_configuration
```

Configuration should support:

```text
Feature Flags
Billing Policies
Proration Policy
Dunning Policy
Usage Thresholds
Token TTL
Webhook Retry Policy
Trial Defaults
Subscription Defaults
```

Configuration precedence should be explicit:

```text
System Default
    ↓
Environment
    ↓
Platform Configuration
    ↓
Tenant Configuration
    ↓
Subscription-specific Policy
```

---

# 60. Security Requirements

Mandatory:

- TLS everywhere
- Strong service authentication
- Short-lived access tokens
- Key rotation
- Secret manager / KMS
- Database encryption where required
- Encryption at rest
- Encryption in transit
- Input validation
- Rate limiting
- Audit logging
- Least privilege
- Tenant isolation
- Signed webhooks
- Signed entitlement tokens
- Replay protection
- Idempotency
- Dependency scanning
- Container image scanning
- Secure CI/CD

---

# 60.1 Data Retention, Privacy & Compliance

Licensing stores commercial and lightly personal data (billing account name/email, customer references, IP/user-agent in audit). Treat it as a first-class enterprise concern.

## Data classification

```text
COMMERCIAL   plans, prices, subscriptions, entitlements, usage
PII          billing_name, billing_email, customer contact references
SENSITIVE    signing private keys (never in DB), webhook secrets, hardware fingerprints
OPERATIONAL  audit, events, outbox/inbox
```

## Retention

- Define a retention period per class rather than deleting on request. Commercial records that back finalized invoices, tax, or revenue recognition are typically retained for a statutory period (often 7–10 years) — coordinate with `accounting`.
- High-volume operational data (`license_meter_event`, `license_usage_ledger`, `license_webhook_delivery`, `license_audit`) needs an explicit archival + partition-drop policy (see §58); never unbounded growth.
- Idempotency keys, inbox rows, and delivered webhooks expire on a short TTL once their window closes.

## Privacy / right-to-erasure

- Support erasure requests via **crypto-shredding or pseudonymization** (drop/rotate the field key, replace `billing_email`/`billing_name` with tombstones) rather than physically deleting rows that back financial history — preserve referential and audit integrity.
- Never place PII in URLs, logs, traces, cache keys, event payloads, or entitlement tokens (tokens carry ids + capabilities only, §19).
- Minimize: store a **hash** of hardware fingerprints, not the raw identifier (§ schema on-premise); reference customers by id, not by copying personal data.

## Compliance posture

- **PCI DSS:** licensing stays out of PCI scope by design — it stores only provider references (`payment_intent_id`, `provider_reference`), never PAN/card data (§27, §32). Keep it that way.
- **SOC 2 / ISO 27001:** the audit trail (§48) must be append-only and complete for commercial and access-control changes; secret access and key rotation are audited.
- **Data residency:** where a tenant requires regional storage, the partition/tenant-routing strategy (§58) must be able to pin a tenant's rows to a region without domain-model changes.

---

# 61. Error Model

Use stable machine-readable error codes.

Example:

```json
{
  "error": {
    "code": "SUBSCRIPTION_PLAN_NOT_AVAILABLE",
    "message": "The selected plan is not available.",
    "correlation_id": "corr_123",
    "details": {}
  }
}
```

Do not expose raw SQLAlchemy/PostgreSQL errors to clients.

---

# 62. State Machine Rules

State transitions must be explicitly validated.

Example:

```text
DRAFT → TRIALING
DRAFT → ACTIVE
TRIALING → ACTIVE
TRIALING → CANCELED
ACTIVE → PAST_DUE
ACTIVE → PAUSED
ACTIVE → CANCELED
PAUSED → ACTIVE
PAST_DUE → ACTIVE
PAST_DUE → SUSPENDED
SUSPENDED → CANCELED
```

Do not allow arbitrary:

```text
PATCH status = ACTIVE
```

from the API.

Use domain commands.

---

# 63. Command Pattern

Prefer domain commands such as:

```text
CreateSubscription
ActivateSubscription
ChangeSubscriptionPlan
AddSubscriptionAddon
RemoveSubscriptionAddon
PauseSubscription
ResumeSubscription
CancelSubscription
ReactivateSubscription
RecordUsage
CompileEntitlement
ApplyCoupon
FinalizeBillingDocument
```

This keeps business rules out of controllers.

---

# 64. Application Layer

Example:

```python
class ChangeSubscriptionPlanHandler:
    async def handle(self, command):
        subscription = await self.repo.get(command.subscription_id)

        subscription.change_plan(
            new_plan_version_id=command.plan_version_id,
            effective_type=command.effective_type,
        )

        await self.repo.save(subscription)

        await self.outbox.publish(
            SubscriptionUpdated(...)
        )
```

The controller should remain thin.

---

# 65. Domain Layer

Business invariants belong in the domain.

Examples:

```text
A canceled subscription cannot be paused.
A retired plan version cannot be newly subscribed to.
A coupon cannot exceed redemption limits.
A hard quota cannot be exceeded.
A finalized billing document cannot be modified.
A published plan version cannot be mutated.
```

Do not put all business rules inside FastAPI route functions.

---

# 66. Repository Layer

Repositories abstract persistence:

```python
class SubscriptionRepository(Protocol):
    async def get(self, subscription_id: UUID) -> Subscription:
        ...

    async def save(self, subscription: Subscription) -> None:
        ...
```

Domain logic should not depend directly on SQLAlchemy query construction.

---

# 67. API Response Versioning

Use:

```text
/api/v1
```

When a breaking contract change is required:

```text
/api/v2
```

Do not silently change the meaning of an existing API field.

---

# 68. Testing Strategy

Required test layers:

```text
Unit Tests
Integration Tests
Repository Tests
API Tests
Contract Tests
Event Tests
Concurrency Tests
Security Tests
Performance Tests
End-to-End Tests
```

Critical scenarios:

```text
Duplicate subscription request
Duplicate usage event
Concurrent quota consumption
Concurrent coupon redemption
Plan version immutability
Proration accuracy
Entitlement precedence
Token signature validation
Signing key rotation
Webhook retry
Outbox recovery
Inbox duplicate event
Tenant isolation
```

---

# 69. Contract Testing

For platform-to-platform integrations, maintain contracts for:

```text
p02_organization
p01_identity
payment
tax
accounting
```

Example:

```text
Licensing → Payment
PaymentRequested.v1

Payment → Licensing
PaymentSucceeded.v1
PaymentFailed.v1
```

Contracts should be versioned and tested independently.

---

# 70. Example End-to-End Subscription Flow

```text
1. Tenant selects Enterprise.
2. Checkout validates plan version.
3. Add-ons are selected.
4. Coupon is validated.
5. Tax calculation is requested.
6. Commercial total is calculated.
7. Payment request is emitted.
8. Payment platform processes payment.
9. Payment success event is received.
10. Subscription becomes ACTIVE.
11. SubscriptionActivated event is created.
12. Entitlement compilation is requested.
13. Worker compiles effective features and limits.
14. Entitlement JWT is generated.
15. Gateway receives the new entitlement.
16. Tenant can access licensed modules.
17. Usage is recorded.
18. Metering calculates overage.
19. Billing cycle closes.
20. Commercial billing document is finalized.
21. Accounting consumes the financial event.
```

---

# 71. Plan Upgrade Flow

```text
Tenant
  ↓
POST /subscriptions/{id}/change-plan
  ↓
Validate new plan
  ↓
Calculate proration
  ↓
Request payment if required
  ↓
Payment succeeded
  ↓
Update subscription
  ↓
Outbox
  ↓
SubscriptionUpdated
  ↓
Entitlement Worker
  ↓
New entitlement
  ↓
Signed JWT
```

If the payment fails, the subscription should not be moved into an unauthorized upgraded state unless the configured business policy explicitly permits it.

---

# 72. Plan Downgrade Flow

Recommended default:

```text
Downgrade Request
  ↓
Validate
  ↓
Schedule END_OF_TERM
  ↓
Keep current entitlement
  ↓
Billing Period Ends
  ↓
Apply New Plan
  ↓
Compile Entitlement
```

This avoids unexpected immediate capability loss.

---

# 73. Subscription Cancellation

Support:

```text
Immediate Cancellation
End-of-Term Cancellation
Scheduled Cancellation
```

Cancellation should preserve:

```text
Previous Plan
Entitlement Version
Billing Period
Cancellation Reason
Actor
Timestamp
Commercial Snapshot
```

---

# 74. Entitlement Revocation

When access must be removed immediately:

```text
Subscription Suspended
    ↓
Entitlement Revocation Event
    ↓
Compile new entitlement
    ↓
Invalidate / expire token
    ↓
Gateway rejects access
```

For security-critical revocations, do not rely solely on a long-lived JWT.

Use short TTL tokens and/or a revocation/version check.

---

# 75. Data Ownership Matrix

| Domain | Owner |
|---|---|
| Tenant master | `p02_organization` |
| Identity | `p01_identity` |
| Product catalog | `p26_licensing` |
| Subscription | `p26_licensing` |
| Entitlement | `p26_licensing` |
| Usage | `p26_licensing` |
| Payment execution | `payment` |
| Tax calculation | `tax` |
| General Ledger | `accounting` |
| Revenue Recognition | `accounting` |
| Financial Statements | `accounting` |
| ERP business data | Individual ERP modules |

---

# 76. Golden Rules for Developers

## Rule 1 — No cross-service database joins

Always use:

```text
API
Events
External IDs
```

## Rule 2 — No accounting inside licensing

Do not create:

```text
GL accounts
Journal entries
Ledger postings
Revenue recognition
```

## Rule 3 — No payment-provider logic inside licensing

Use:

```text
payment
```

## Rule 4 — No tax engine inside licensing

Use:

```text
tax
```

## Rule 5 — Never use float for money

Use:

```text
Decimal / minor units
```

## Rule 6 — Published commercial versions are immutable

Use new versions.

## Rule 7 — Runtime authorization must be fast

Use:

```text
Materialized Entitlement
Signed Token
Redis
```

## Rule 8 — Usage is immutable

Use:

```text
Adjustment
Reversal
```

instead of deleting history.

## Rule 9 — Every external command should be idempotent

Use:

```text
Idempotency-Key
```

## Rule 10 — Every important transaction should produce an event safely

Use:

```text
Transactional Outbox
```

---

# 77. Recommended Initial Implementation Order

Build the platform in stages.

### Phase 1 — Foundation

```text
Core
Database
Enums
Tenant Context
Security Context
Base Models
Audit
Outbox
Inbox
Idempotency
```

### Phase 2 — Commercial Definition

```text
Catalog
Products
Features
Plans
Plan Versions
Prices
Currencies
Units
Add-ons
```

### Phase 3 — Subscription

```text
Subscriptions
Items
Changes
Schedules
Trials
Contracts
Snapshots
```

### Phase 4 — Entitlement

```text
Entitlements
Overrides
Sources
Compilation
Signing Keys
Signed Tokens
```

### Phase 5 — Metering

```text
Meters
Usage Events
Usage Periods
Usage Ledger
Quota
Rate Limits
Overage
Rating
```

### Phase 6 — Commercial Billing

```text
Checkout
Quotes
Discounts
Coupons
Billing Cycles
Commercial Billing Documents
Proration
Dunning
```

### Phase 7 — Integrations

```text
payment
tax
accounting
p02_organization
p01_identity
```

### Phase 8 — Advanced Licensing

```text
Marketplace
On-Premise Licensing
Hardware Binding
Key Rotation
Revocation
```

---

# 78. Production Readiness Checklist

Before production:

- [ ] PostgreSQL migrations tested
- [ ] Tenant isolation tested
- [ ] Authorization tested
- [ ] Published plan immutability tested
- [ ] Subscription state machine tested
- [ ] Proration tested with Decimal arithmetic
- [ ] Coupon race conditions tested
- [ ] Usage idempotency tested
- [ ] Quota concurrency tested
- [ ] Entitlement precedence tested
- [ ] JWT signature tested
- [ ] Signing key rotation tested
- [ ] Outbox recovery tested
- [ ] Inbox duplicate handling tested
- [ ] Webhook retries tested
- [ ] Dead-letter handling tested
- [ ] Payment integration contract tested
- [ ] Tax integration contract tested
- [ ] Accounting event contract tested
- [ ] Audit immutability tested
- [ ] Database backup tested
- [ ] Restore procedure tested
- [ ] Monitoring configured
- [ ] Alerting configured
- [ ] Security scanning enabled
- [ ] Load testing completed
- [ ] Disaster recovery tested

---

# 79. Reference Runtime Architecture

```text
                         ┌─────────────────────┐
                         │     p02_organization        │
                         └──────────┬──────────┘
                                    │
                                    ▼
┌───────────────┐          ┌─────────────────────┐
│ p01_identity  │─────────▶│   p26_licensing     │
└───────────────┘          │                     │
                           │ Catalog             │
                           │ Plans               │
                           │ Pricing             │
                           │ Subscription        │
                           │ Entitlement         │
                           │ Metering            │
                           │ Billing Coordination │
                           └───────┬─────────────┘
                                   │
                     ┌─────────────┼─────────────┐
                     ▼             ▼             ▼
              payment      tax    accounting
```

Runtime:

```text
Client
  ↓
API Gateway
  ↓
Identity Validation
  ↓
Entitlement Token
  ↓
Feature Check
  ↓
Rate Limit
  ↓
Quota
  ↓
ERP Service
  ↓
Usage Event
  ↓
p26_licensing
```

---

# 80. Final Architecture Principle

The platform should remain intentionally narrow:

```text
p26_licensing
=
Commercial Subscription
+
Entitlement
+
Metering
+
Licensing
```

Not:

```text
Licensing
+
Payment Processor
+
Tax Engine
+
Accounting
+
ERP
```

The long-term architecture is therefore:

```text
p02_organization
    ↓
p01_identity
    ↓
p26_licensing
    ├── payment
    ├── tax
    └── accounting
```

Each platform owns its own data, business rules, APIs, events, migrations, and operational lifecycle.

The integration layer connects them without merging their domains.

---

## Appendix A — Primary Tables

> **Schema alignment (resolved).** The production schema (`LICENSING_SCHEMA.md`) is the authoritative source for table definitions. The seven previously guide-only tables are now reconciled:
>
> Defined as real tables in **Schema §63.1**:
> - `license_contract_entitlement` — contract-granted entitlements feeding compilation (§14).
> - `license_subscription_pause` — pause history (replaces relying only on `paused_at`/`resume_at`).
> - `license_cancellation` — cancellation record (immediate / end-of-term / scheduled).
> - `license_usage_reset` — meter/quota reset events (§21).
> - `license_discount_rule` — structured discount rules alongside `license_coupon_condition`.
> - `license_quote_conversion` — quote→subscription conversion record (§28).
>
> **Folded (not a table):**
> - `license_entitlement_source` — entitlement provenance lives on `license_entitlement_feature.source_type` / `source_id`. A materialized entitlement is a hot read path, so provenance is denormalized rather than joined. Treat the `license_entitlement_source` name in §16 as those columns, not a table.
>
> The two documents now agree.

```text
CATALOG
license_product
license_product_category
license_product_version
license_feature
license_feature_dependency
license_module
license_module_feature
license_unit
license_currency

PRICING
license_price
license_price_tier
license_price_component

PLANS
license_plan
license_plan_version
license_plan_version_feature

ADDONS
license_addon
license_addon_version
license_addon_feature

SUBSCRIPTION
license_subscription
license_subscription_item
license_subscription_change
license_subscription_schedule
license_subscription_schedule_phase
license_subscription_trial
license_trial_policy
license_contract
license_contract_line
license_contract_entitlement
license_subscription_pause
license_cancellation
license_subscription_snapshot

ENTITLEMENT
license_entitlement
license_entitlement_feature
license_entitlement_limit
license_entitlement_override
license_entitlement_source
license_entitlement_compilation
license_entitlement_token
license_signing_key

METERING
license_meter
license_meter_event
license_usage_period
license_usage
license_usage_ledger
license_usage_reset
license_usage_overage
license_usage_rating
license_rate_limit_policy

COMMERCIAL BILLING
license_billing_account
license_billing_cycle
license_invoice
license_invoice_line
license_proration
license_customer_balance
license_balance_transaction
license_dunning_policy
license_dunning_case
license_payment_reference
license_refund_reference
license_chargeback_reference
license_external_reference
license_reconciliation_run
license_reconciliation_item

CHECKOUT / QUOTES
license_checkout
license_checkout_item
license_quote
license_quote_line
license_quote_conversion

DISCOUNTS
license_discount
license_discount_rule
license_coupon
license_coupon_condition
license_coupon_usage

MARKETPLACE
license_marketplace_publisher
license_marketplace_app
license_marketplace_version
license_marketplace_purchase
license_marketplace_installation
license_marketplace_permission

ON-PREMISE
license_deployment
license_license_key
license_activation
license_hardware_binding
license_revocation

INTEGRATION
license_webhook_endpoint
license_webhook_delivery
license_event
license_outbox_event
license_inbox_event
license_idempotency_key

OPERATIONS
license_audit
license_notification_template
license_notification_delivery
license_configuration
```

---

## Appendix B — Non-Negotiable Service Boundaries

```text
┌────────────────────┐
│ p02_organization           │
│ Master Data        │
└────────────────────┘

┌────────────────────┐
│ p01_identity       │
│ Authentication/IAM │
└────────────────────┘

┌────────────────────┐
│ p26_licensing      │
│ Subscription       │
│ Entitlement        │
│ Metering           │
│ Licensing          │
└────────────────────┘

┌────────────────────┐
│ payment        │
│ Payment Processing │
└────────────────────┘

┌────────────────────┐
│ tax            │
│ Tax Engine         │
└────────────────────┘

┌────────────────────┐
│ accounting     │
│ Finance / GL / AR  │
└────────────────────┘
```

This separation is the foundation for independently scalable, replaceable, testable, and internationally extensible NEXTERP platforms.
