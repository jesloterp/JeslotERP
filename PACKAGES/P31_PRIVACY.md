# Privacy (`p31_privacy`)

**Package:** `p31_privacy`  
**Schema:** `privacy`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Privacy operations: data subjects, consent, policies, legal holds, and data-subject-request workflow with erase adapters.

## 2. Responsibilities

- Own the `privacy` persistence schema and the `p31_privacy` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Subjects
- Consent withdraw
- Policies
- Holds release
- DSR open / advance / operations
- Internal purposes list
- Corpus crawlers and erase adapters for identity, partner, and media

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Data subject**
- **Purpose**
- **Consent**
- **Policy**
- **Legal hold**
- **DSR case**
- **DSR operation**

## 5. Major Capabilities

- Subjects
- Consent withdraw
- Policies
- Holds release
- DSR open / advance / operations
- Internal purposes list
- Corpus crawlers and erase adapters for identity, partner, and media

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

- Subjects
- Consent / policy / hold
- DSR cases / operations
- Outbox

Public API resource groups:

- Subjects / consents / policies / holds / DSR
- Internal purposes

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p19_audit`

### Additional verified runtime coupling

- Erase / crawl adapters for identity, business partner, and media

### Recommended future dependency

- Purpose-create HTTP and fuller DSR operations UI

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Compliance operators
- partner erasure flows

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `privacy.subject.created`
- `privacy.consent.withdrawn`
- `privacy.hold.created`
- `privacy.dsr.opened`
- `privacy.dsr.advanced`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

No additional public environment knobs are documented.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- No blind delete
- Holds block progression
- privacy permissions

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Erase port
- Corpus crawler registry

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `privacy` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Purpose-create HTTP is not complete
- Fuller DSR operations UI contract is pending
- Not every domain has an erase adapter yet

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

GDPR-style rights are a platform workflow, not a DELETE FROM partners.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
