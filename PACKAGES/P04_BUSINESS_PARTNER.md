# Business Partner (`p04_business_partner`)

**Package:** `p04_business_partner`  
**Schema:** `bp`  
**Layer:** Platform Foundation  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Enterprise party / counterparty system of record: customers, vendors, and other partner roles.

## 2. Responsibilities

- Own the `bp` persistence schema and the `p04_business_partner` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Partner lifecycle and roles
- Addresses, contacts, tax, and bank satellites
- Sites, relationships, and hierarchy
- Lookups and categories
- KYC onboarding
- Match and merge
- Change requests
- Credit and risk
- Compliance / blacklist
- Erasure / export / import
- Metadata-bound list sort and filter contract for the partner pilot

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Partner**
- **Partner role**
- **Site**
- **Match candidate**
- **Golden record / merge**
- **KYC case**
- **Change request**
- **Credit profile**
- **Legal hold / erasure**

## 5. Major Capabilities

- Partner lifecycle and roles
- Addresses, contacts, tax, and bank satellites
- Sites, relationships, and hierarchy
- Lookups and categories
- KYC onboarding
- Match and merge
- Change requests
- Credit and risk
- Compliance / blacklist
- Erasure / export / import
- Metadata-bound list sort and filter contract for the partner pilot

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

- Partner core
- Satellites
- Financial / credit
- Governance
- Match / privacy
- Lookups
- Outbox

Public API resource groups:

- Partners
- Roles
- Satellites
- KYC
- Match / merge
- Compliance
- Credit
- Erasure
- Settings
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`
- `p03_configuration`

### Additional verified runtime coupling

- Number allocation via p07_number_series
- Record visibility via p33_sharing
- Lifecycle hooks via p29_extensibility

### Recommended future dependency

- p05_metadata layouts (pilot entity bp.partner is seeded)
- p08_file_media attachments
- p10_process for change-request approvals
- p31_privacy for erasure

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Sales, Purchase, CRM, Finance (planned)
- Licensing
- Process
- Privacy adapters

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `bp.partner.created`
- `bp.partner.updated`
- `bp.partner.activated`
- `bp.partner.deactivated`
- `bp.partner.blacklisted`
- `bp.address.changed`
- `bp.partner.role_assigned`
- `bp.partner.credit_changed`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Partner codes bind to number-series objects. Hook keys are declared, not arbitrary.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Namespaced partner permissions
- Row-level isolation
- Bank reveal gated
- Record sharing
- Redaction / masking

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Extensibility hook points on save
- Metadata binding contract for bp.partner
- Sharing grants on create

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `bp` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Steward / golden-record operations workflow is still pending
- Production label withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

No sales, purchase, treasury, or CRM module can exist without a governed party master.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
