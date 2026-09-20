# Licensing (`p26_licensing`)

**Package:** `p26_licensing`  
**Schema:** `licensing`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Commercial catalog, subscription, entitlement, usage, checkout, dunning, marketplace, and on-prem license artifacts.

## 2. Responsibilities

- Own the `licensing` persistence schema and the `p26_licensing` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Product / plan / feature / module catalog
- Subscription lifecycle
- Entitlement compile, token, and check
- Usage ingest, rating, overage
- Checkout, quotes, invoices, dunning, contracts
- Marketplace / on-prem surfaces
- Platform versus tenant owner houses
- Internal runtime guard
- Payment gateway ports exist and fail-closed without credentials

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Product**
- **Plan version**
- **Subscription**
- **Entitlement**
- **Usage meter**
- **Checkout**
- **Invoice**
- **Dunning case**
- **Contract**
- **Marketplace app**
- **On-prem license**

## 5. Major Capabilities

- Product / plan / feature / module catalog
- Subscription lifecycle
- Entitlement compile, token, and check
- Usage ingest, rating, overage
- Checkout, quotes, invoices, dunning, contracts
- Marketplace / on-prem surfaces
- Platform versus tenant owner houses
- Internal runtime guard
- Payment gateway ports exist and fail-closed without credentials

## 6. Public Architecture

```text
HTTP / internal API
        ↓
Application services (commands, queries, ports)
        ↓
Domain concepts and policies
        ↓
Adapters (persistence, optional providers, outbox)
```

Conceptual tables (purpose only):

- Subscription
- Entitlement
- Usage
- Billing / checkout
- Marketplace / on-prem
- Outbox

Public API resource groups:

- Catalog / subscriptions / entitlements / usage / checkout / dunning / houses
- Internal guard

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p03_configuration`
- `p04_business_partner`
- `p12_feature`
- `p13_event_bus`
- `p14_messaging`
- `p16_cache`
- `p17_scheduler`
- `p19_audit`
- `p21_monitoring`

### Additional verified runtime coupling

- Startup bridges to event bus, jobs, cache, scheduler, audit, and metrics

### Recommended future dependency

- Payment service provider in production
- p12_feature flag–SKU compile (product decision pending)

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Frontend entitlement hydrate
- future module guards
- internal runtime guard

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `licensing.plan.published`
- `licensing.checkout.confirmed`
- `licensing.invoice.finalized`
- `licensing.dunning.step`
- licensing.subscription lifecycle types via relay

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Payment adapter selection is environment-specific; credentials are never documented here.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- House-separated permissions
- SKU guard
- Token introspection
- Quota enforcer

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Payment-gateway port
- Entitlement compiler

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `licensing` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Live payment providers are environment-blocked
- Flag–SKU compile binding is blocked pending product decision
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Module access and usage must be entitlement-backed before commercial SaaS launch.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
