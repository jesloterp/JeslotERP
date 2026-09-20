# Security Architecture

Public view only. Threat-model worksheets, key material, and private findings
are excluded.

## Control planes

| Plane | Package | Owns | Does not own |
|---|---|---|---|
| Identity | `p01_identity` | AuthN, sessions, RBAC, MFA, OAuth | Org master data, KMS catalog |
| Security ops | `p28_security` | Key metadata, rotation, posture, WAF profiles | Login, secret **values** |
| Configuration secrets | `p03_configuration` | Secret **references** and ciphertext metadata | Key custody |
| Record ACL | `p33_sharing` | Teams, rules, grants, evaluate | Tenant RLS, role catalog |
| Audit | `p19_audit` | Immutable trail, holds, export | Technical logs |
| Privacy | `p31_privacy` | Consent, DSR, erase orchestration | Blind row delete |
| API product | `p22_api` | Key hashes, rate limits, products | Business authorization |

## Authentication (current)

Implemented kernel capabilities include password session flows, refresh,
logout, MFA, WebAuthn, OAuth / OIDC client surfaces, and service accounts.

Production identity-provider activation and complete SAML are **not** claimed.

## Authorization

- Namespaced permissions seeded into identity.
- Fail-closed HTTP dependencies.
- Field-level security **descriptors** live in metadata; identity / callers enforce.
- Sharing evaluate is additive record ACL on selected domains.
- Licensing runtime guard is a commercial gate, not a replacement for RBAC.

## Tenancy and isolation

See [MULTI_TENANCY.md](MULTI_TENANCY.md). RLS is the database primitive.
Gateway checks resolve company / branch context.

## Secrets and keys

- Public APIs do not return secret plaintext.
- KMS adapters other than local development remain provider-pending.
- This document does not list environment variable names that would aid attack.

## Expressions, hooks, and AI

- Metadata and rules: allow-listed AST, no `eval`, no arbitrary SQL.
- Extensibility: allow-list + isolated worker + timeout.
- AI: guardrails, tool confirm / deny, quota events. AI must not bypass process.

## Auditability

Security-relevant identity events and platform audit ingest exist. SIEM
forwarding is a port and is fail-closed without an endpoint.

## Honest gaps

| Gap | Status |
|---|---|
| Signed platform threat review | OPEN |
| Production IdP / JWKS | Environment-blocked |
| Enterprise KMS / HSM | Environment-blocked |
| Soak of gapless numbering under concurrency | Environment-blocked |
| Production label | Withheld |

## Publication rule

If a control is uncertain, describe it as a capability area — never paste
private class names, algorithms, or credentials.
