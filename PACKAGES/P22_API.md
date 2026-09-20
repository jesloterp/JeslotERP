# API (`p22_api`)

**Package:** `p22_api`  
**Schema:** `api`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Inbound API product plane: catalog, OpenAPI specs, products / plans, keys, rate limits, portal, bulk, and composite.

## 2. Responsibilities

- Own the `api` persistence schema and the `p22_api` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Services, operations, specs, versions
- Products, plans, subscriptions
- Hashed API keys
- Rate policies and snapshots
- Analytics and portal pages
- Bulk and composite gateways
- Internal key verify and rate-limit check
- Optional distributed rate limiter when reachable

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **API service / operation**
- **OpenAPI spec / version**
- **Product / plan**
- **Subscription**
- **API key (hash only)**
- **Rate policy**
- **Gateway snapshot**
- **Bulk job**
- **Composite subrequest**

## 5. Major Capabilities

- Services, operations, specs, versions
- Products, plans, subscriptions
- Hashed API keys
- Rate policies and snapshots
- Analytics and portal pages
- Bulk and composite gateways
- Internal key verify and rate-limit check
- Optional distributed rate limiter when reachable

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

- Services / operations
- Specs / versions
- Products / plans / subscriptions
- Credentials / limits / policies
- Analytics / governance

Public API resource groups:

- Catalog / specs / products / keys / limits / portal / bulk / composite
- Internal verify

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p12_feature`
- `p16_cache`

### Additional verified runtime coupling

- Gateway snapshot via p16_cache SDK

### Recommended future dependency

- All HTTP platforms
- p23_integration as the outbound counterpart
- p19_audit

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Integration
- AI (declared)
- external integrators

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `api.spec.published`
- `api.product.activated`
- `api.key.issued`
- `api.key.revoked`
- `api.limit.violated`
- `api.snapshot.published`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Rate-limiter attach is optional at startup.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Only key hashes are stored
- IP allow-list / CORS / idempotency policies
- api permissions

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Rate-limiter port
- Snapshot cache
- Pack / CI gate hooks

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `api` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- OData / GraphQL are explicitly out of current demand
- Edge-gateway deployment topology is NOT_VERIFIED
- Durable bulk-job runner beyond process-local is pending

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Partners and devices integrate through versioned API products, not ad-hoc table access.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
