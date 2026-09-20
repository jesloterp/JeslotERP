# Sharing (`p33_sharing`)

**Package:** `p33_sharing`  
**Schema:** `sharing`  
**Layer:** Platform Foundation  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Record-level access: teams, hierarchy, sharing rules, grants, and evaluate. Distinct from tenant row-level security and from role permissions.

## 2. Responsibilities

- Own the `sharing` persistence schema and the `p33_sharing` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Teams and members
- Hierarchy
- Rules activate
- Grants revoke
- Public and internal evaluate
- Database-backed record-access façade

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Team**
- **Hierarchy**
- **Sharing rule**
- **Grant**
- **Evaluation result**

## 5. Major Capabilities

- Teams and members
- Hierarchy
- Rules activate
- Grants revoke
- Public and internal evaluate
- Database-backed record-access façade

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

- Teams
- Hierarchy
- Rules
- Grants
- Outbox

Public API resource groups:

- Teams / hierarchy / rules / grants / evaluate
- Internal evaluate

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p02_organization`

### Additional verified runtime coupling

- Wired on partner, media, document, and process list / access paths

### Recommended future dependency

- Evaluate on remaining domain list / GET surfaces as modules appear

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- p04_business_partner
- p08_file_media
- p09_document
- p10_process

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `sharing.team.created`
- `sharing.member.added`
- `sharing.rule.activated`
- `sharing.grant.created`
- `sharing.grant.revoked`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Empty-grant evaluate semantics are behavioral, not secret configuration.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Fail-closed when grants exist
- Owner grant on create for the partner pilot
- Does not expand IAM into this bounded context

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Resource-type constants
- Evaluate façade

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `sharing` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Not every domain list / GET is wired yet
- Does not replace tenant RLS or IAM roles

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Account teams and plant-limited documents need record ACL on top of role permissions.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
