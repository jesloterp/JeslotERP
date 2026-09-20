# Notification (`p15_notification`)

**Package:** `p15_notification`  
**Schema:** `notification`  
**Layer:** Infrastructure  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Omnichannel notification plane: templates, routing, preferences, quiet hours, delivery ledger, inbox, and digests.

## 2. Responsibilities

- Own the `notification` persistence schema and the `p15_notification` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Send, preview, and request APIs
- In-app inbox
- Preferences, devices, and quiet hours
- Templates
- Topics, routes, and providers
- Suppression, digest, webhooks, packs
- Internal dispatch
- Channel adapter ports exist; live SMTP / SMS / WhatsApp fail-closed without credentials

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Topic**
- **Template version / locale / channel**
- **Send request**
- **Delivery**
- **Provider**
- **Preference / consent**
- **Quiet hours**
- **Inbox**
- **Digest**

## 5. Major Capabilities

- Send, preview, and request APIs
- In-app inbox
- Preferences, devices, and quiet hours
- Templates
- Topics, routes, and providers
- Suppression, digest, webhooks, packs
- Internal dispatch
- Channel adapter ports exist; live SMTP / SMS / WhatsApp fail-closed without credentials

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

- Topics / providers
- Templates
- Preferences / devices
- Requests / deliveries
- Inbox
- Digest / governance

Public API resource groups:

- Send / preview
- Inbox
- Preferences
- Templates
- Catalog
- Ops
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p06_localization`
- `p14_messaging`

### Additional verified runtime coupling

- Dispatch jobs via p14_messaging

### Recommended future dependency

- p08_file_media for attachments
- p10_process for task notices
- p03_configuration secret refs

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Process (recommended)
- Output handoff
- Monitoring alerts (recommended)
- future business modules

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `notify.template.activated`
- `notify.delivery.delivered`
- `notify.delivery.bounced`
- `notify.inbox.read`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Provider secrets are referenced through the configuration plane.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- notify permissions including critical-send
- Webhook signature validation
- Suppression and quiet hours

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Channel adapter port
- Enqueue bridge
- Packs

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `notification` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Live delivery providers are environment-blocked
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Approvals, dunning, and operational alerts must be consented, localized, and auditable.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
