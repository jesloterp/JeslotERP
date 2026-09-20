# Output (`p32_output`)

**Package:** `p32_output`  
**Schema:** `output`  
**Layer:** Platform Services  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Output determination, templates, render, jobs, and spool. Distinct from media storage, document identity, and notification delivery.

## 2. Responsibilities

- Own the `output` persistence schema and the `p32_output` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Templates, versions, activate
- Determinations
- Determine and render
- Jobs retry and spool
- Internal determine / render
- TEXT renderer is implemented
- PDF renderer exists as a port and is fail-closed without engine dependencies

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Output template**
- **Template version**
- **Determination rule**
- **Render job**
- **Spool entry**

## 5. Major Capabilities

- Templates, versions, activate
- Determinations
- Determine and render
- Jobs retry and spool
- Internal determine / render
- TEXT renderer is implemented
- PDF renderer exists as a port and is fail-closed without engine dependencies

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

- Templates / versions
- Determinations
- Jobs / spool
- Outbox

Public API resource groups:

- Templates / determinations / render / jobs / spool
- Internal determine / render

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p06_localization`

### Additional verified runtime coupling

- Rendered artifacts can store via p08_file_media
- Handoff to p15_notification
- Partner print seed exists

### Recommended future dependency

- Staging verification of PDF engine dependencies

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Business Partner print pilot
- future invoices, orders, and statutory forms

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `output.template.created`
- `output.template.version.activated`
- `output.determination.created`
- `output.job.rendered`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

PDF engine selection is environment-specific.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- output permissions
- RLS
- Rendered content via media references

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Renderer ports
- Handoff ports to media and notification

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `output` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Production PDF engine verification is environment-blocked
- Broad business-document print catalog is not started

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Which form prints for which company, language, and channel is determination — not if/else in each module.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
