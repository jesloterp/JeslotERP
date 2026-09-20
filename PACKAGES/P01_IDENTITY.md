# Identity (`p01_identity`)

**Package:** `p01_identity`  
**Schema:** `identity`  
**Layer:** Platform Foundation  
**Implementation status:** `IMPLEMENTED`  
**Kernel posture:** SoR-Live  
**Production label:** FUTURE

This document is a public architecture specification. It does not include source code, credentials, or private algorithms.

## 1. Purpose

Central identity, authentication, authorization, session, and security-primitive authority for the platform.

## 2. Responsibilities

- Own the `identity` persistence schema and the `p01_identity` module boundary.
- Expose a versioned HTTP surface under the platform API conventions.
- Seed a namespaced permission catalog consumed by `downstream platforms`.
- Emit integration events through an outbox; do not import another package's ORM models.
- Remain fail-closed when optional providers are not configured.

## 3. Scope

### In scope

- Password authentication, refresh, and logout
- Registration and verification flows
- Multi-factor authentication and WebAuthn
- OAuth / OIDC clients, scopes, and well-known discovery
- Session and device management
- Users, roles, permissions, and RBAC catalog
- Service accounts
- Security event recording
- Internal organization-context APIs

### Out of scope

- Business ledgers, stock, tax calculation, and industry documents (those belong to `business/bNN_*`).
- Another package's system of record.
- Production-only vendor credentials and private deployment topology.

## 4. Core Concepts

- **User**
- **Session**
- **Role**
- **Permission**
- **Tenant assignment**
- **MFA credential**
- **WebAuthn credential**
- **OAuth / OIDC client**
- **Service account**
- **Device**

## 5. Major Capabilities

- Password authentication, refresh, and logout
- Registration and verification flows
- Multi-factor authentication and WebAuthn
- OAuth / OIDC clients, scopes, and well-known discovery
- Session and device management
- Users, roles, permissions, and RBAC catalog
- Service accounts
- Security event recording
- Internal organization-context APIs

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

- Users and credentials
- Sessions and tokens
- Roles, permissions, and assignments
- MFA / WebAuthn
- OAuth clients
- Devices
- Security / outbox plumbing

Public API resource groups:

- Public authentication and current-user surfaces
- Admin IAM
- OAuth / OIDC
- MFA / WebAuthn
- Sessions and devices
- RBAC catalog
- Internal org-context

## 7. Dependencies

### Actual dependency (declared module plugin)

- None declared (foundational).

### Additional verified runtime coupling

- Optional field-level security descriptors from p05_metadata on profile surfaces
- Organization existence checks via gateway (not ORM import)

### Recommended future dependency

- p02_organization for tenant/company context resolution
- p28_security for enterprise key management beyond identity primitives

Do not treat recommended lines as implemented wiring.

## 8. Consumers

- All authenticated platforms
- Organization onboarding
- Licensing houses
- Privacy subject adapters

## 9. Events

Representative public event names observed in the platform (not an exhaustive private catalog):

- iam.user.logged_in
- iam.user.logout_all
- iam.user.locked
- iam.context.switched
- iam.role.assigned

End-to-end delivery through `p13_event_bus` is the intended bus; some packages still persist a local outbox. Treat cross-bus consumption as `NOT_VERIFIED` unless a consumer is listed above.

## 10. Configuration

Token lifetimes, lockout policy, and OIDC client settings are environment configuration — not published here.

Public documentation does not list secret names, private URLs, or credential material.

## 11. Security Considerations


- Fail-closed authorization
- Lockout and reuse detection
- PKCE-capable OAuth paths
- Permission-catalog seeding for other platforms

- Tenant isolation is expected on tenant-owned tables.
- Cross-schema foreign keys are forbidden; references are UUID + gateway / event.
- Optional providers must fail closed rather than invent success.

## 12. Extension Points

- Permission catalog as the shared authorization vocabulary
- OAuth scopes
- Gateway-based org context

## 13. Current Implementation Status

| Dimension | Status |
|---|---|
| Package present | Yes |
| Module plugin | Yes |
| HTTP surface | Yes |
| Persistence schema `identity` | Yes |
| Automated tests | Yes |
| Kernel | SoR-Live |
| Feature completeness | `IMPLEMENTED` |
| Production | FUTURE |

Known gaps:

- Production IdP / JWKS activation remains environment-dependent
- SAML assertion consumer is not a complete enterprise SSO product
- Formal threat-model sign-off is still open
- Registry Production label is withheld

## 14. Planned Capabilities

- Close provider and soak gaps listed above.
- Keep the registry Production label withheld until soak, threat review, and live-provider evidence exist.
- Expand consumers as business modules appear — without inventing new platform numbers.

## 15. Business Impact

Every business transaction, approval, and audit record depends on a trusted actor and permission set.

## 16. Open-Source Considerations

- Publish this specification, not the private implementation, until an intentional source release occurs.
- Keep allow-lists, fail-closed providers, and permission names as the public contract.
- Do not publish seed data that contains customer, tenant, or credential material.
