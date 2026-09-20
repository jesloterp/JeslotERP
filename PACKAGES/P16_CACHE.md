# Cache (`p16_cache`)

**Package:** `p16_cache`  
**Schema:** `cache`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Shared cache control plane: namespaces, TTL / tag policies, invalidation, warmup, and L1 / L2 backends.

## 2. Responsibilities

- Own the `cache` persistence schema and the `p16_cache` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Namespace and policy catalog
- Invalidate and purge
- Warmup plans and runs
- Backend catalog
- Internal get / set / get-or-load
- Application SDK
- Catalog is persistence-backed
- Distributed L2 attaches when available; otherwise remains pending

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Namespace**
- **Key template**
- **TTL policy**
- **Tag**
- **Invalidation request**
- **Warmup plan**
- **Backend binding**
- **Sensitivity / buffer class**

## 5. Major Capabilities

- Namespace and policy catalog
- Invalidate and purge
- Warmup plans and runs
- Backend catalog
- Internal get / set / get-or-load
- Application SDK
- Catalog is persistence-backed
- Distributed L2 attaches when available; otherwise remains pending

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

- Namespaces / policies / backends / tags
- Invalidate / warmup
- Governance

Public API resource groups:

- Catalog
- Invalidate
- Warmup
- Backends
- Ops
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p03_configuration`

### Additional verified runtime coupling

- None beyond declared module dependencies.

### Recommended future dependency

- Read-heavy platforms
- p14_messaging / p17_scheduler for warmup jobs

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- API gateway snapshots
- Dashboard widget cache
- Licensing entitlement cache

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- Invalidate-bus endpoints exist; a full public event catalog is NOT_VERIFIED

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

L2 attach is optional at startup.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- cache permissions
- Sensitivity forbidden errors
- RLS
- Admin purge permissions

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- L2 backend port
- SDK
- Invalidate bus
- Packs

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `cache` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Distributed L2 is provider-pending
- Production stampede policy at scale is not claimed

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Effective metadata, entitlements, and rate snapshots must not each invent cache keys.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
