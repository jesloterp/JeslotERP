# JeslotERP Licensing Platform — Developer Integration Guide (V2)

**Version:** 2.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — product/subscription/entitlement HTTP persists on Postgres; empty list is `[]`. Live PSP is `PROVIDER_PENDING` (pytest stub is a test double only). Not Production.  
**Package:** `platforms.p26_licensing`  
**PostgreSQL schema:** `licensing`  
**Table prefix:** `license_`  
**Depends on:** `p01_identity`, `p02_organization`, `p03_configuration`, `p04_business_partner`, `p12_feature`  
**Integrates with:** `p13_event_bus`, `p14_messaging`, `p15_notification`, `p16_cache`, `p17_scheduler`, `p19_audit`, `p21_monitoring`, `p22_api`, `p23_integration` (payment/tax connectors), `p27_ai` (quota/entitlement hooks); logical `payment` / `tax` / `accounting` when assigned  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`V2_LICENSING_SCHEMA.md`](V2_LICENSING_SCHEMA.md) · [`V2_LICENSING_API.md`](V2_LICENSING_API.md)  
**Legacy deep refs (v1 encyclopedia):** [`LICENSING_GUIDE.md`](LICENSING_GUIDE.md) · [`LICENSING_SCHEMA.md`](LICENSING_SCHEMA.md) · [`LICENSING_API.md`](LICENSING_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **2.0 Advanced** | **2026-09-09** | Peer-aligned triad format: registry deps 01–04+12, p12 flag split, `/api/v1/licensing` mount, StandardResponse, peer matrix (p14/p15/p19/p22/p23/p27). Preserves commercial SoR depth; does not replace v1 ORM dump. |
| **2.0 SoR-Live** | **2026-09-12** | TASK-SOR-022: product/subscription/entitlement HTTP Postgres-first; empty `[]`; `require_licensing_access`. Stripe/Razorpay live classes fail `PROVIDER_PENDING` (no invented PSP ids). |
| 1.x | 2026-09-08 | Original encyclopedic Guide/Schema/API (renumbered from p04 → p26). |

---

## 1. Purpose (enterprise)

`p26_licensing` is JeslotERP’s **subscription, entitlement, and commercial metering control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Subscription Billing / S/4 commercial entitlements** — products, plans, contracts, usage  
- **Microsoft Dynamics 365 / Commerce licensing patterns** — SKUs, entitlements, seats/meters  
- **Salesforce Edition / add-on / usage entitlements** — compile → enforce → meter  
- **Stripe Billing / Zuora-class subscription engines** — checkout, proration, dunning (without owning GL)  

It is **not** “a boolean `is_premium` column.” It answers:

> What did this tenant purchase, what is active now, and what capabilities/limits may they use?

### Owns

| Domain | Examples |
|---|---|
| Catalog | Products, modules, commercial features |
| Plans / add-ons | Immutable plan versions, prices |
| Subscriptions | Lifecycle, items, changes, schedules, trials |
| Contracts | Commitments, contract entitlements |
| Entitlements | Compile, overrides, signed tokens |
| Metering | Meters, usage ledger, quotas, overage, rating |
| Commerce ops | Checkout, quotes, coupons, proration |
| Billing coordination | Invoices meta, dunning state, payment refs |
| Marketplace / on-prem | Installs, license keys, activation |
| Governance | Webhooks, outbox, packs |

### Does **not** own

| Concern | Owner |
|---|---|
| Feature flags / rollout / kill switches | `p12_feature` |
| IAM permissions / roles | `p01_identity` |
| Tenant/company master | `p02_organization` |
| Long-lived settings values | `p03_configuration` |
| Partner master | `p04_business_partner` |
| Payment capture / PCI | `payment` (logical) via `p23_integration` |
| Tax calculation ledger | `tax` (logical) |
| GL / revenue recognition | `accounting` (logical) |
| User email/SMS delivery | `p15_notification` |
| Public API product keys | `p22_api` |

### Critical splits

| | **Licensing (p26)** | **Feature (p12)** | **Identity (p01)** |
|---|---|---|---|
| Question | Are they **entitled** (paid/contract)? | Is the capability **rolled out**? | Are they **allowed** (RBAC)? |
| Example | `module.sales.advanced` | `sales.order.bulk_import.v2` | `document.create` |
| Cadence | Commercial contract | Release / experiment | Security policy |

**Runtime rule (normative):**

```text
effective = entitled(p26) AND flag_on(p12) AND permitted(p01)
```

Never treat a license feature as a LaunchDarkly-style flag, or a flag as a paid SKU.

---

## 2. Architectural position

```text
Catalog (product/plan/price)
        │
   Checkout / Quote / Contract
        │
   Subscription (active)
        │
   Entitlement compile ──► signed token / cache (p16)
        │
   Runtime check ◄── platforms (AND p12 AND p01)
        │
   Usage meters ──► rating / overage ──► invoice meta
        │
   payment/tax (logical) via p23 · notify p15 · audit p19 · jobs p14
```

**Hard rules**

1. No cross-schema FKs — UUID refs + gateways only.  
2. Published plan/price versions are **immutable**.  
3. Entitlements are **compiled** artifacts, not hand-edited production rows (overrides are explicit).  
4. Tenant identity from verified JWT/security context — never trust client `X-Tenant-Id` alone.  
5. Money uses `Numeric(18,6)` (see Schema).  
6. Payment/tax/GL are **coordination refs**, not ledgers inside licensing.  
7. Usage ingest is **idempotent**.  
8. Register HTTP surface in **p22** under `/api/v1/licensing`.

---

## 3. Advanced design principles

1. **Commercial definition ≠ runtime enforcement** — catalog/plan vs compiled entitlement.  
2. **Version everything sellable** — plan_version, addon_version, price effective windows.  
3. **Compile on change** — subscription change → recompile → invalidate token/cache.  
4. **Seat + meter + boolean** feature kinds.  
5. **Precedence** — override > contract > addon > plan (document explicitly in compile).  
6. **Signed entitlement JWT** — short-lived; rotate signing keys.  
7. **Proration as pure calc** — accounting books elsewhere.  
8. **Dunning state machine** — soft block vs hard suspend policy.  
9. **On-prem licenses** — issue/activate/heartbeat separate from SaaS subscription.  
10. **Marketplace installs** bind to subscription entitlements.  
11. **CQRS** — commerce commands vs hot entitlement queries.  
12. **Outbox** — subscription/entitlement/usage domain events.  
13. **Workers via p14** — compile, rating, dunning, webhooks.  
14. **Clocks via p17** — renewals, trial end, dunning steps.  
15. **Observability** — compile latency, check RPS, quota breaches → p21.  
16. **Packs** — seed product SKUs / trial plans.  
17. **Optimistic concurrency** — integer `version` + `If-Match`.  
18. **AI quotas** — p27 may meter against p26 entitlement limits.

---

## 4. Core concepts

### 4.1 Product / plan / price

Sellable catalog. Plans publish immutable versions that freeze feature grants and limits.

### 4.2 Subscription

Tenant commercial agreement instance: status lifecycle (trial → active → past_due → canceled…).

### 4.3 Entitlement compilation

Materialized view of features/limits for a subscription at time T; source provenance on feature rows (`source_type` / `source_id`) — **not** a separate `license_entitlement_source` table.

### 4.4 Runtime check

`check-feature` / `check-limit` / token introspection — hot path, cached.

### 4.5 Meter / usage

Event ingest → period aggregate → quota enforce → optional overage rating.

### 4.6 Checkout / quote

Pre-subscription commercial carts; quote conversion creates subscription.

### 4.7 Billing coordination

Invoice documents for subscription commerce; `payment_reference` points outward.

---

## 5. Integration patterns

| Concern | Integration |
|---|---|
| AuthN/Z admin APIs | p01 JWT + `licensing.*` permissions |
| Tenant/company | p02 UUID refs |
| Billing profile prefs | p03 optional keys |
| Partner / sold-to | p04 soft refs |
| Flag AND gate | p12 evaluate after entitled |
| Domain events | p13 |
| Compile/rating/dunning jobs | p14 |
| Trial ending / invoice ready | p15 |
| Entitlement token cache | p16 |
| Renewals / dunning schedule | p17 |
| Commerce mutations audit | p19 |
| Check latency / errors | p21 |
| Route catalog | p22 |
| Payment/tax adapters | p23 |
| AI token caps | p27 entitlement_hook |

---

## 6. Security

### Permissions (representative)

| Code | Use |
|---|---|
| `licensing.catalog.read` | Products/plans |
| `licensing.catalog.manage` | Catalog admin |
| `licensing.subscription.read` | Read subscriptions (manage implies read) |
| `licensing.subscription.manage` | Subscribe / change-plan / cancel |
| `licensing.entitlement.read` | Checks / tokens |
| `licensing.entitlement.compile` | Force recompile |
| `licensing.usage.ingest` | Meter events |
| `licensing.usage.read` | Usage reports |
| `licensing.checkout.manage` | Checkout/quotes |
| `licensing.billing.manage` | Invoices/dunning meta |
| `licensing.onprem.manage` | On-prem keys |
| `licensing.admin` | Packs, keys, purge |
| `licensing.*` | Wildcard |

### RLS

FORCE RLS on tenant subscriptions, entitlements, usage, checkouts, invoices, on-prem activations.  
Global catalog (products/plans/prices) readable when published; manage = admin.

---

## 7. Module layout

```text
platforms/p26_licensing/
  application/
    services/
      catalog.py                 # GUIDE name → catalog_store facade
      catalog_store.py           # dual-run fields + catalog CRUD + billing stubs
      store_support.py
      subscription_lifecycle.py
      entitlement_compiler.py
      entitlement_token.py
      usage_ingest.py
      quota_enforcer.py
      rating.py
      onprem.py
      runtime_gate.py            # effective = entitled AND flag AND permitted
      durable.py                 # HTTP dual-write
    permissions/catalog.py
  domain/…
  infrastructure/
    http/… persistence/… cache/ payment_gateway/ tax_gateway/
  tests/unit/…
```

HTTP stays thin: routers call `get_catalog_store()` / `durable` / `subscription_house`. Checkout/dunning money stays on the facade until Phase G. CQRS command/query split is not required for SoR-Live.

---

## 8. Domain events

| Event | When |
|---|---|
| `licensing.plan.published` | Catalog |
| `licensing.subscription.created` / `changed` / `canceled` | Lifecycle |
| `licensing.entitlement.compiled` | Compile |
| `licensing.usage.recorded` / `quota.exceeded` | Meters |
| `licensing.checkout.confirmed` | Commerce |
| `licensing.invoice.finalized` | Billing meta |
| `licensing.dunning.step` / `suspended` | Collections |
| `licensing.onprem.activated` | On-prem |

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs — V2 triad is SSoT; v1 encyclopedia retained as ORM/API history (REST archived, HYG-015) |
| P1 | Skeleton, permissions, catalog |
| P2 | Plans/prices immutable publish |
| P3 | Subscriptions + lifecycle |
| P4 | Entitlement compile + check + token |
| P5 | Usage ingest + quotas |
| P6 | Checkout/quotes/coupons |
| P7 | Invoice meta + dunning + payment refs |
| P8 | On-prem + marketplace + packs |
| P9 | Registry → **Live** |

---

## 10. Definition of Done (enterprise)

Evidence: `platforms/p26_licensing/tests/unit/api/test_licensing_phase_i.py`. Billing money (G) is still deferred.

- [x] `effective = entitled AND flag AND permitted` documented in platform SDKs (`runtime_gate.effective_and`, API §4.3, frontend `isEffectiveCheck`)
- [x] No cross-schema FKs (UUID columns only; no `ForeignKey`)
- [x] Plan version immutable after publish (`409 CONFLICT`)
- [x] Usage ingest idempotent (`duplicate: true`)
- [x] Quota exceed returns deterministic deny + code (`429 QUOTA_EXCEEDED`)
- [x] Tenant isolation on subscriptions/usage (JWT tenant; other tenant `404`)
- [x] Signing keys rotatable without downtime (previous kids still verify)
- [x] Payment/tax/GL not stored as ledgers here (`license_payment_reference` + `PROVIDER_PENDING`)
- [x] Routes under `/api/v1/licensing` registered in p22 (`service_key=licensing`)  

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Use p12 flags as SKUs | Commercial features in p26 |
| Edit live entitlement rows | Recompile + explicit overrides |
| FK to identity/org tables | UUID + gateway |
| Put Stripe charge logic in domain services | payment via p23 |
| Trust `X-Tenant-Id` alone | Verified security context |
| Mount bare `/api/v1/products` in monolith | `/api/v1/licensing/products` |

---

## 12. Related documents

- Schema V2: [`V2_LICENSING_SCHEMA.md`](V2_LICENSING_SCHEMA.md)  
- API V2: [`V2_LICENSING_API.md`](V2_LICENSING_API.md)  
- Feature: [`../12_feature/FEATURE_GUIDE.md`](../12_feature/FEATURE_GUIDE.md)  
- API platform: [`../22_api/API_GUIDE.md`](../22_api/API_GUIDE.md)  
- AI quotas: [`../27_ai/AI_GUIDE.md`](../27_ai/AI_GUIDE.md)  
- v1 deep Guide: [`LICENSING_GUIDE.md`](LICENSING_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
