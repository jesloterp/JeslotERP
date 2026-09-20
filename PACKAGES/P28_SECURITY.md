# Security (`p28_security`)

**Package:** `p28_security`  
**Schema:** `security`  
**Layer:** Platform Foundation  
**Implementation status:** `PARTIALLY_IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Security operations plane: KMS abstraction, key lifecycle, policy, posture, and WAF profiles. Not login (p01) and not secret values (p03).

## 2. Responsibilities

- Own the `security` persistence schema and the `p28_security` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `p01_identity`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Provider list / test
- Key CRUD, rotate, retire (no secret material on GET)
- Policies and posture run
- WAF profiles
- Security event list
- Internal key ensure / version
- Local-development provider is implemented
- Cloud KMS / HSM adapters exist as pending ports

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **Provider**
- **Key**
- **Key version**
- **Rotation job**
- **Policy**
- **Posture check**
- **WAF profile**
- **Security event**

## 5. Major Capabilities

- Provider list / test
- Key CRUD, rotate, retire (no secret material on GET)
- Policies and posture run
- WAF profiles
- Security event list
- Internal key ensure / version
- Local-development provider is implemented
- Cloud KMS / HSM adapters exist as pending ports

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

- Providers
- Keys / versions / rotation
- Policies
- Posture
- WAF profiles
- Events
- Outbox

Public API resource groups:

- Providers / keys / policies / posture / WAF / events
- Internal keys

## 7. Dependencies

### Actual dependency (declared module plugin)

- `p01_identity`
- `p03_configuration`

### Additional verified runtime coupling

- Consumed by p30_alm for package signing

### Recommended future dependency

- Enterprise KMS / HSM in production

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- Configuration secret wrap
- ALM signing

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- `security.key.created`
- `security.key.rotated`

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Provider selection is environment-specific; credentials are never published here.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- No secret GET
- security permissions
- RLS

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- KMS port and provider registry

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `security` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `PARTIALLY_IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Cloud KMS / HSM live credentials are environment-blocked
- SIEM sink is explicitly out of scope (belongs to p19_audit)

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Key custody and rotation must be a platform concern before regulated deployments.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
