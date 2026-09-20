# JeslotERP Licensing Platform — Production Schema (V2 Advanced)

**Version:** 2.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — `license_product` / `license_subscription` / `license_entitlement` / `license_payment_reference` are the HTTP commercial ledger. PSP refs only. Not Production.  
**Package:** `platforms.p26_licensing`  
**PostgreSQL schema:** `licensing`  
**Table prefix:** `license_`  
**Companion:** [`V2_LICENSING_GUIDE.md`](V2_LICENSING_GUIDE.md) · [`V2_LICENSING_API.md`](V2_LICENSING_API.md)  
**ORM / DDL encyclopedia (v1):** [`LICENSING_SCHEMA.md`](LICENSING_SCHEMA.md) — full SQLAlchemy models, constraints, `Numeric(18,6)`, validated mapper set (~94 tables).

> Runtime models: `platforms/p26_licensing/infrastructure/persistence/models/`.  
> V2 is the **navigable inventory + key contracts**. v1 remains the **line-by-line ORM authority**.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `licensing` (never `p26` / `p04`) |
| Tables | `license_*` |
| Soft delete | `status` incl. `DELETED` + partial unique indexes |
| Money / qty | `Numeric(18,6)` |
| Optimistic lock | integer `version` (`If-Match`) |
| Cross-schema | UUID refs only (tenant, company, partner, user) |
| RLS | FORCE on tenant commercial/usage/entitlement tables |
| Provenance | `source_type` / `source_id` on entitlement features — **no** `license_entitlement_source` table |

---

## 2. Complete table inventory (**93 physical tables**)

> Count excludes folded logical name `license_entitlement_source`. Plumbing included below.

### 2.1 Catalog (9)

| # | Table | Purpose |
|---|---|---|
| 1 | `license_currency` | Currencies |
| 2 | `license_product_category` | Categories |
| 3 | `license_product` | Products |
| 4 | `license_product_version` | Product versions |
| 5 | `license_feature` | Commercial features |
| 6 | `license_feature_dependency` | Feature deps |
| 7 | `license_module` | Modules |
| 8 | `license_module_feature` | Module ↔ feature |
| 9 | `license_unit` | Units (seat, GB, call…) |

### 2.2 Pricing (3)

| # | Table | Purpose |
|---|---|---|
| 10 | `license_price` | Prices |
| 11 | `license_price_tier` | Tiered pricing |
| 12 | `license_price_component` | Components |

### 2.3 Plans & add-ons (6)

| # | Table | Purpose |
|---|---|---|
| 13 | `license_plan` | Plans |
| 14 | `license_plan_version` | Immutable versions |
| 15 | `license_plan_version_feature` | Grants/limits |
| 16 | `license_addon` | Add-ons |
| 17 | `license_addon_version` | Addon versions |
| 18 | `license_addon_feature` | Addon grants |

### 2.4 Subscription & contracts (14)

| # | Table | Purpose |
|---|---|---|
| 19 | `license_subscription` | Subscriptions |
| 20 | `license_subscription_item` | Line items |
| 21 | `license_subscription_change` | Change history |
| 22 | `license_subscription_schedule` | Schedules |
| 23 | `license_subscription_schedule_phase` | Phases |
| 24 | `license_subscription_trial` | Trials |
| 25 | `license_trial_policy` | Trial policies |
| 26 | `license_contract` | Contracts |
| 27 | `license_contract_line` | Contract lines |
| 28 | `license_contract_entitlement` | Contract grants |
| 29 | `license_subscription_pause` | Pause history |
| 30 | `license_cancellation` | Cancellations |
| 31 | `license_subscription_snapshot` | Snapshots |
| 32 | `license_billing_account` | Billing profiles |

### 2.5 Entitlements & tokens (7)

| # | Table | Purpose |
|---|---|---|
| 33 | `license_entitlement` | Compiled header |
| 34 | `license_entitlement_feature` | Features + provenance cols |
| 35 | `license_entitlement_limit` | Limits |
| 36 | `license_entitlement_override` | Explicit overrides |
| 37 | `license_entitlement_compilation` | Compile runs |
| 38 | `license_entitlement_token` | Issued tokens meta |
| 39 | `license_signing_key` | JWT signing keys |

### 2.6 Metering & usage (9)

| # | Table | Purpose |
|---|---|---|
| 40 | `license_meter` | Meter defs |
| 41 | `license_meter_event` | Raw events |
| 42 | `license_usage_period` | Periods |
| 43 | `license_usage` | Aggregates |
| 44 | `license_usage_ledger` | Ledger lines |
| 45 | `license_usage_reset` | Resets |
| 46 | `license_usage_overage` | Overage |
| 47 | `license_usage_rating` | Rating results |
| 48 | `license_rate_limit_policy` | Rate policies |

### 2.7 Commercial billing coordination (14)

| # | Table | Purpose |
|---|---|---|
| 49 | `license_billing_cycle` | Cycles |
| 50 | `license_invoice` | Commercial invoices |
| 51 | `license_invoice_line` | Lines |
| 52 | `license_proration` | Proration calcs |
| 53 | `license_customer_balance` | Balance |
| 54 | `license_balance_transaction` | Balance txs |
| 55 | `license_dunning_policy` | Dunning policies |
| 56 | `license_dunning_case` | Cases |
| 57 | `license_payment_reference` | Payment refs |
| 58 | `license_refund_reference` | Refund refs |
| 59 | `license_chargeback_reference` | Chargebacks |
| 60 | `license_external_reference` | External ids |
| 61 | `license_reconciliation_run` | Reconcile runs |
| 62 | `license_reconciliation_item` | Items |

### 2.8 Checkout, quotes, discounts (10)

| # | Table | Purpose |
|---|---|---|
| 63 | `license_checkout` | Checkouts |
| 64 | `license_checkout_item` | Items |
| 65 | `license_quote` | Quotes |
| 66 | `license_quote_line` | Quote lines |
| 67 | `license_quote_conversion` | Quote → sub |
| 68 | `license_discount` | Discounts |
| 69 | `license_discount_rule` | Rules |
| 70 | `license_coupon` | Coupons |
| 71 | `license_coupon_condition` | Conditions |
| 72 | `license_coupon_usage` | Redemptions / usage |

### 2.9 Marketplace (6)

| # | Table | Purpose |
|---|---|---|
| 73 | `license_marketplace_publisher` | Publishers |
| 74 | `license_marketplace_app` | Apps |
| 75 | `license_marketplace_version` | App versions |
| 76 | `license_marketplace_purchase` | Purchases |
| 77 | `license_marketplace_installation` | Installs |
| 78 | `license_marketplace_permission` | App permissions |

### 2.10 On-premise (5)

| # | Table | Purpose |
|---|---|---|
| 79 | `license_deployment` | Deployments |
| 80 | `license_license_key` | Issued keys |
| 81 | `license_activation` | Activations |
| 82 | `license_hardware_binding` | Hardware binds |
| 83 | `license_revocation` | Revocations |

### 2.11 Integration & plumbing (9+)

| # | Table | Purpose |
|---|---|---|
| 84 | `license_webhook_endpoint` | Webhooks |
| 85 | `license_webhook_delivery` | Deliveries |
| 86 | `license_event` | Domain event log meta |
| 87 | `license_outbox_event` | Outbox |
| 88 | `license_inbox_event` | Inbox |
| 89 | `license_idempotency_key` | Idempotency |
| 90+ | `license_audit`, config, extras | See v1 for full set to ~94 |

**Normative:** For every column, FK, CHECK, and partial unique, **[`LICENSING_SCHEMA.md`](LICENSING_SCHEMA.md) wins**.

---

## 3. Enumerations (selected)

| Enum | Values (representative) |
|---|---|
| `subscription_status` | `TRIAL`, `ACTIVE`, `PAST_DUE`, `PAUSED`, `CANCELED`, `EXPIRED` |
| `plan_lifecycle` | `DRAFT`, `PUBLISHED`, `RETIRED` |
| `feature_value_type` | `BOOLEAN`, `LIMIT`, `SEAT`, `METERED` |
| `entitlement_source_type` | `PLAN`, `ADDON`, `CONTRACT`, `OVERRIDE`, `TRIAL`, `PROMO` |
| `usage_event_status` | `ACCEPTED`, `DUPLICATE`, `REJECTED` |
| `dunning_state` | `OPEN`, `REMINDED`, `SOFT_BLOCK`, `SUSPENDED`, `CLOSED` |
| `invoice_status` | `DRAFT`, `FINALIZED`, `VOID`, `PAID_REF` |

---

## 4. Key table contracts

### 4.1 `license_plan_version`

| Column | Notes |
|---|---|
| `plan_id` | Parent |
| `version_number` | Unique per plan |
| `lifecycle` | DRAFT→PUBLISHED→RETIRED |
| `published_at` | Set once |
| `checksum` | Optional content seal |

**Immutable after PUBLISHED** — changes require new version.

### 4.2 `license_subscription`

| Column | Notes |
|---|---|
| `tenant_id` | RLS |
| `subscription_number` | Unique per tenant |
| `plan_version_id` | Current |
| `status` | Lifecycle |
| `billing_account_id` | Profile |
| `contract_id` | Optional |
| `current_period_start/end` | Window |
| `version` | Optimistic lock |

### 4.3 `license_entitlement_feature`

| Column | Notes |
|---|---|
| `entitlement_id` | Parent |
| `feature_id` | Catalog feature |
| `is_enabled` | Boolean grant |
| `limit_value` | Nullable |
| `source_type` / `source_id` | Provenance (folded “source” table) |

### 4.4 `license_meter_event`

| Column | Notes |
|---|---|
| `tenant_id` | RLS |
| `subscription_id` | |
| `meter_id` | |
| `idempotency_key` | Unique per tenant |
| `quantity` | Numeric |
| `occurred_at` | Event time |

### 4.5 `license_payment_reference`

| Column | Notes |
|---|---|
| `provider` | `stripe`, `razorpay`, … |
| `external_payment_id` | Soft ref |
| `status` | Coordination only |
| `invoice_id` | Optional |

No card PANs / charge ledgers here.

---

## 5. Entitlement compile model

```text
inputs:
  plan_version features
  + addon_version features
  + contract_entitlements
  + overrides
  + trial policy grants
→ precedence resolve
→ write entitlement + feature/limit rows
→ record compilation
→ issue/rotate token meta
→ invalidate p16 cache
```

---

## 6. RLS summary

| Class | Policy |
|---|---|
| Global catalog/currency/plans | Read published; manage admin |
| Tenant subscriptions/items/changes | FORCE `tenant_id` |
| Entitlements/tokens/usage | FORCE `tenant_id` |
| Checkout/quotes/invoices | FORCE `tenant_id` |
| On-prem activations | FORCE `tenant_id` |

---

## 7. Seed minimum

1. Currencies `INR`, `USD`  
2. Product `jesloterp.saas` + modules finance/inventory  
3. Plans `starter`, `growth`, `enterprise` (published v1)  
4. Meters `api.calls`, `ai.tokens` (for p27 hook)  
5. Trial policy 14-day  
6. Permissions `licensing.*`  
7. Pack `core.saas.v1`  

---

## 8. ER overview

```text
product ── plan ── plan_version ── features/limits
                └── addon_version ── features
subscription ── items/changes/schedules/trials
        └── entitlement ── features/limits/overrides/tokens
        └── meters ── events/usage/overage/rating
checkout/quote ──► subscription
invoice meta ── payment/refund/chargeback refs
contract ── contract_entitlement
onprem / marketplace / webhooks / outbox
```

---

## 9. Implementation notes

1. Prefer v1 `GlobalEntity` / `TenantScopedEntity` bases.  
2. Partial uniques: `WHERE status <> 'DELETED'`.  
3. Intra-licensing FKs OK; never FK into `identity`/`organization` schemas.  
4. Split model packages: `catalog`, `pricing`, `subscription`, `entitlement`, `usage`, `billing`, `checkout`, `onprem`, `plumbing`.
