# Organization (`p02_organization`)

**Package:** `p02_organization`  
**Schema:** `org`  
**Layer:** Platform Foundation  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Tenant, company, branch, fiscal, and organizational-structure system of record. Separate from identity.

## 2. Responsibilities

- Own the `org` persistence schema and the `p02_organization` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Tenant provision and onboarding
- Company and branch hierarchy
- Departments, designations, and employees
- Addresses and contacts
- Fiscal periods
- Cost and profit centers
- Warehouses and locations
- Calendars and holidays
- Tax registrations
- Units of measure and FX conversion
- Organization tree and context APIs

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Tenant**
- **Company**
- **Branch**
- **Department**
- **Employee**
- **Fiscal year / period**
- **Warehouse / location**
- **Working calendar**
- **Unit of measure**
- **Foreign-exchange rate**
- **Cost / profit center**

## 5. Major Capabilities

- Tenant provision and onboarding
- Company and branch hierarchy
- Departments, designations, and employees
- Addresses and contacts
- Fiscal periods
- Cost and profit centers
- Warehouses and locations
- Calendars and holidays
- Tax registrations
- Units of measure and FX conversion
- Organization tree and context APIs

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

- Platform masters (country, currency, UoM)
- Tenant-scoped structure
- Fiscal and logistics masters
- Onboarding and outbox

Public API resource groups:

- Tenants
- Companies
- Branches
- Masters
- Fiscal
- Warehouses
- Employees
- Measures
- Tree
- Bulk
- Internal

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`

### Additional verified runtime coupling

- IAM permission seeding
- Onboarding coordination with identity

### Recommended future dependency

- p03_configuration for operational settings
- p06_localization for locale defaults

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Business Partner
- Configuration
- Metadata
- Number Series
- Licensing
- Sharing
- all tenant-scoped platforms

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `org.company.created`
- `org.branch.created`
- `org.tenant.provisioned`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Tenant branding and operational org settings live in organization tables, distinct from p03 setting values.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Organization-scoped permissions
- Tenant and company access dependencies
- Row-level isolation on org data

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Internal APIs for other platforms
- Permission catalog seed

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `org` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Plant / sales-organization encyclopedia not added
- Production soak and threat review open

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

All ERP postings, inventory, and documents are scoped to company, branch, and fiscal context.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
