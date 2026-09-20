# JeslotERP Identity Platform — Complete Advanced Developer & Implementation Guide

**Document:** Identity / IAM Platform Developer Guide  
**Version:** 3.1  
**Last reviewed:** 2026-09-12  
**Target:** Python 3.12+, FastAPI, SQLAlchemy 2.x, PostgreSQL 16, Redis 7, Celery, OAuth 2.1 / OIDC, MFA, WebAuthn, SSO  
**Architecture:** Multi-tenant, event-driven, modular, production-ready  
**Package:** `platforms.p01_identity`  
**Status:** **SoR-Live** — runtime package is `platforms.p01_identity` (registry / SCHEMA). Not `identity.iam`. Not Production.  
**Threat model:** [`IDENTITY_THREAT_MODEL.md`](IDENTITY_THREAT_MODEL.md) — UNSIGNED; gate stays OPEN.  
**FLS:** p05 declarations on `identity.user` are enforced on profile GET/PATCH (no declaration → skip).

---

# 1. Purpose

The JeslotERP Identity Platform is the central security authority for the entire ERP.

It owns:

- User identity
- Authentication
- Passwords
- Email verification
- OTP
- WhatsApp OTP
- MFA
- Passkeys / WebAuthn
- Sessions
- Access tokens
- Refresh tokens
- OAuth 2.1
- OpenID Connect
- RBAC
- Permissions
- Roles
- Security policies
- Tenant/company/branch access assignments
- Service accounts
- SSO / federation
- Security events
- Audit events

It does **not** own:

- Tenant master records
- Company master records
- Branch master records
- Department master records
- Finance business data
- Sales business data
- Inventory business data

Those belong to their respective bounded contexts.

---

# 2. Core Architectural Rule

The most important separation is:

```text
ORG
    Owns what organizations exist.

IDENTITY / IAM
    Owns who can access those organizations and what they can do.
```

Therefore:

```text
ORG
├── tenant
├── company
├── branch
└── department

IAM
├── user
├── tenant assignment
├── company assignment
├── branch assignment
├── department assignment
├── role
└── permission
```

A user is a **global identity**.

Do NOT make the global user itself tenant-owned.

Do NOT put these on `identity_iam_user`:

```text
tenant_id
company_id
branch_id
```

Instead use assignment tables.

---

# 3. Multi-Tenant Identity Model

A single user can belong to multiple tenants.

Example:

```text
User: Chetan
│
├── Tenant A
│   ├── Company A1
│   │   ├── Branch A1-B1
│   │   └── Branch A1-B2
│   └── Company A2
│
└── Tenant B
    └── Company B1
        └── Branch B1-B1
```

The hierarchy is:

```text
User
 ↓
Tenant Assignment
 ↓
Company Assignment
 ↓
Branch Assignment
 ↓
Department Assignment
```

Assignments are access records.

The organization itself remains owned by `ORG`.

---

# 4. Context Switching

A user can have multiple operational contexts.

A context is:

```text
tenant_id
company_id
branch_id
```

Example current context:

```json
{
  "tenant_id": "T1",
  "company_id": "C1",
  "branch_id": "B1"
}
```

When the user switches:

```text
Tenant A → Tenant B
```

do not trust frontend-supplied tenant IDs as authorization.

The Identity Platform must verify the assignment and issue a new context-bound access token.

Recommended endpoint:

```http
POST /api/v1/auth/context/switch
```

The server verifies:

```text
User → Tenant
User → Company
User → Branch
```

Then creates a new token containing the new context.

---

# 5. Token Rule

Never use request headers as the authoritative tenant/company/branch security mechanism.

Do not rely on:

```http
X-Tenant-ID
X-Company-ID
X-Branch-ID
```

The authoritative context comes from the verified access token.

Example:

```json
{
  "iss": "https://idp.jesloterp.example",
  "sub": "USER_UUID",
  "aud": "jeslot-api",
  "jti": "TOKEN_UUID",
  "tenant_id": "TENANT_UUID",
  "company_id": "COMPANY_UUID",
  "branch_id": "BRANCH_UUID",
  "session_id": "SESSION_UUID",
  "scope": "openid profile finance",
  "amr": ["pwd", "mfa"],
  "iat": 1770000000,
  "exp": 1770003600
}
```

Keep JWTs small.

Do not place the entire permission tree inside the JWT.

---

# 6. Request Context

Every authenticated request should produce a trusted application context.

Recommended object:

```python
from dataclasses import dataclass
from uuid import UUID


@dataclass(frozen=True)
class RequestContext:
    user_id: UUID
    tenant_id: UUID | None
    company_id: UUID | None
    branch_id: UUID | None
    session_id: UUID | None
    scopes: frozenset[str]
```

The flow is:

```text
Bearer Token
    ↓
Signature verification
    ↓
Issuer validation
    ↓
Audience validation
    ↓
Expiration validation
    ↓
Session validation
    ↓
RequestContext
```

Business modules receive `RequestContext`.

---

# 7. Technology Rules

Use:

- Python 3.12+
- `uv`
- FastAPI
- SQLAlchemy 2.x
- PostgreSQL 16
- Alembic
- Redis 7
- Celery
- Pydantic v2
- Argon2id for password hashing
- JWT with asymmetric signing for production
- OAuth 2.1 / OIDC
- WebAuthn / passkeys
- pytest
- mypy

Do not use:

- `pip` as the project dependency manager
- Poetry
- framework code inside the domain
- SQLAlchemy inside domain
- FastAPI inside domain
- database access inside domain
- business logic inside FastAPI routers

---

# 8. Project Structure

Use this complete structure:

Runtime import path is `platforms.p01_identity`. The tree is GUIDE layout, not a second package named `identity.iam`.

```text
platforms/p01_identity/
    ├── __init__.py
    ├── module.py
    │
    ├── domain/
    │   ├── __init__.py
    │   │
    │   ├── aggregates/
    │   │   ├── __init__.py
    │   │   ├── user.py
    │   │   ├── session.py
    │   │   ├── role.py
    │   │   ├── oauth_client.py
    │   │   ├── service_account.py
    │   │   └── security_policy.py
    │   │
    │   ├── entities/
    │   │   ├── __init__.py
    │   │   ├── user_profile.py
    │   │   ├── tenant_assignment.py
    │   │   ├── company_assignment.py
    │   │   ├── branch_assignment.py
    │   │   ├── department_assignment.py
    │   │   ├── permission.py
    │   │   ├── resource.py
    │   │   ├── device.py
    │   │   ├── mfa.py
    │   │   ├── webauthn_credential.py
    │   │   └── sso_identity.py
    │   │
    │   ├── value_objects/
    │   │   ├── __init__.py
    │   │   ├── user_id.py
    │   │   ├── tenant_id.py
    │   │   ├── email.py
    │   │   ├── phone_number.py
    │   │   ├── password.py
    │   │   ├── tenant_context.py
    │   │   ├── permission_code.py
    │   │   └── token_id.py
    │   │
    │   ├── events/
    │   │   ├── __init__.py
    │   │   ├── user_created.py
    │   │   ├── user_updated.py
    │   │   ├── user_deleted.py
    │   │   ├── user_logged_in.py
    │   │   ├── user_logged_out.py
    │   │   ├── session_created.py
    │   │   ├── session_revoked.py
    │   │   ├── context_switched.py
    │   │   ├── role_assigned.py
    │   │   ├── role_revoked.py
    │   │   ├── permission_granted.py
    │   │   ├── permission_revoked.py
    │   │   ├── password_changed.py
    │   │   ├── password_reset.py
    │   │   ├── mfa_enabled.py
    │   │   ├── mfa_disabled.py
    │   │   └── security_event_detected.py
    │   │
    │   ├── exceptions/
    │   │   ├── __init__.py
    │   │   ├── user_exceptions.py
    │   │   ├── authentication_exceptions.py
    │   │   ├── authorization_exceptions.py
    │   │   ├── session_exceptions.py
    │   │   ├── tenant_exceptions.py
    │   │   ├── role_exceptions.py
    │   │   └── security_exceptions.py
    │   │
    │   └── services/
    │       ├── __init__.py
    │       ├── authorization_service.py
    │       ├── authentication_service.py
    │       ├── permission_resolver.py
    │       ├── role_resolver.py
    │       ├── tenant_context_service.py
    │       ├── password_policy_service.py
    │       └── security_policy_service.py
    │
    ├── application/
    │   ├── __init__.py
    │   │
    │   ├── commands/
    │   │   ├── __init__.py
    │   │
    │   │   ├── users/
    │   │   │   ├── create_user.py
    │   │   │   ├── update_user.py
    │   │   │   ├── activate_user.py
    │   │   │   ├── deactivate_user.py
    │   │   │   └── delete_user.py
    │   │   │
    │   │   ├── authentication/
    │   │   │   ├── login.py
    │   │   │   ├── logout.py
    │   │   │   ├── refresh_token.py
    │   │   │   ├── change_password.py
    │   │   │   ├── request_password_reset.py
    │   │   │   ├── confirm_password_reset.py
    │   │   │   └── verify_email.py
    │   │   │
    │   │   ├── otp/
    │   │   │   ├── request_otp.py
    │   │   │   ├── verify_otp.py
    │   │   │   └── resend_otp.py
    │   │   │
    │   │   ├── whatsapp/
    │   │   │   ├── request_whatsapp_otp.py
    │   │   │   ├── verify_whatsapp_otp.py
    │   │   │   └── resend_whatsapp_otp.py
    │   │   │
    │   │   ├── context/
    │   │   │   ├── switch_tenant.py
    │   │   │   ├── switch_company.py
    │   │   │   └── switch_branch.py
    │   │   │
    │   │   ├── roles/
    │   │   │   ├── create_role.py
    │   │   │   ├── update_role.py
    │   │   │   ├── delete_role.py
    │   │   │   ├── assign_role.py
    │   │   │   └── revoke_role.py
    │   │   │
    │   │   ├── permissions/
    │   │   │   ├── grant_permission.py
    │   │   │   └── revoke_permission.py
    │   │   │
    │   │   ├── mfa/
    │   │   │   ├── setup_mfa.py
    │   │   │   ├── verify_mfa.py
    │   │   │   ├── disable_mfa.py
    │   │   │   └── regenerate_recovery_codes.py
    │   │   │
    │   │   ├── sessions/
    │   │   │   ├── revoke_session.py
    │   │   │   └── revoke_all_sessions.py
    │   │   │
    │   │   └── organization/
    │   │       ├── assign_tenant.py
    │   │       ├── revoke_tenant.py
    │   │       ├── assign_company.py
    │   │       ├── revoke_company.py
    │   │       ├── assign_branch.py
    │   │       └── revoke_branch.py
    │   │
    │   ├── queries/
    │   │   ├── users/
    │   │   │   ├── get_user.py
    │   │   │   ├── list_users.py
    │   │   │   └── get_user_profile.py
    │   │   │
    │   │   ├── authentication/
    │   │   │   ├── get_current_user.py
    │   │   │   ├── get_current_context.py
    │   │   │   └── get_my_permissions.py
    │   │   │
    │   │   ├── organization/
    │   │   │   ├── get_user_tenants.py
    │   │   │   ├── get_user_companies.py
    │   │   │   ├── get_user_branches.py
    │   │   │   └── get_user_contexts.py
    │   │   │
    │   │   ├── roles/
    │   │   │   ├── get_role.py
    │   │   │   └── list_roles.py
    │   │   │
    │   │   ├── permissions/
    │   │   │   ├── list_permissions.py
    │   │   │   └── resolve_permissions.py
    │   │   │
    │   │   └── sessions/
    │   │       └── list_sessions.py
    │   │
    │   ├── ports/
    │   │   ├── repositories/
    │   │   │   ├── user_repository.py
    │   │   │   ├── session_repository.py
    │   │   │   ├── role_repository.py
    │   │   │   ├── permission_repository.py
    │   │   │   ├── tenant_assignment_repository.py
    │   │   │   ├── oauth_client_repository.py
    │   │   │   ├── refresh_token_repository.py
    │   │   │   ├── mfa_repository.py
    │   │   │   └── device_repository.py
    │   │   │
    │   │   ├── gateways/
    │   │   │   ├── email_gateway.py
    │   │   │   ├── sms_gateway.py
    │   │   │   ├── whatsapp_gateway.py
    │   │   │   ├── organization_gateway.py
    │   │   │   └── event_publisher.py
    │   │   │
    │   │   └── security/
    │   │       ├── token_service.py
    │   │       ├── password_hasher.py
    │   │       ├── encryption_service.py
    │   │       ├── otp_service.py
    │   │       └── authorization_port.py
    │   │
    │   ├── dto/
    │   │   ├── user_dto.py
    │   │   ├── auth_dto.py
    │   │   ├── token_dto.py
    │   │   ├── context_dto.py
    │   │   ├── role_dto.py
    │   │   ├── permission_dto.py
    │   │   └── session_dto.py
    │   │
    │   └── services/
    │       ├── authentication_application_service.py
    │       ├── authorization_application_service.py
    │       ├── context_application_service.py
    │       └── user_application_service.py
    │
    ├── infrastructure/
    │   ├── persistence/
    │   │   ├── models/
    │   │   ├── repositories/
    │   │   ├── mappers/
    │   │   └── unit_of_work.py
    │   │
    │   ├── security/
    │   │   ├── jwt/
    │   │   ├── password/
    │   │   ├── oauth/
    │   │   ├── oidc/
    │   │   ├── mfa/
    │   │   ├── webauthn/
    │   │   └── encryption/
    │   │
    │   ├── messaging/
    │   │   ├── outbox/
    │   │   ├── publishers/
    │   │   └── consumers/
    │   │
    │   ├── gateways/
    │   │   ├── email/
    │   │   ├── sms/
    │   │   ├── whatsapp/
    │   │   └── organization/
    │   │
    │   ├── http/
    │   │   ├── api_v1.py
    │   │   ├── routers/
    │   │   ├── dto/
    │   │   ├── dependencies/
    │   │   └── middleware/
    │   │
    │   └── config/
    │       ├── settings.py
    │       ├── database.py
    │       ├── redis.py
    │       └── security.py
    │
    ├── migrations/
    │   ├── env.py
    │   ├── script.py.mako
    │   └── versions/
    │
    └── tests/
        ├── unit/
        │   ├── domain/
        │   └── application/
        ├── integration/
        │   ├── persistence/
        │   ├── authentication/
        │   ├── authorization/
        │   ├── oauth/
        │   ├── mfa/
        │   ├── whatsapp/
        │   └── sso/
        └── e2e/
            ├── test_login.py
            ├── test_context_switch.py
            ├── test_authorization.py
            ├── test_refresh_rotation.py
            ├── test_mfa.py
            ├── test_whatsapp_otp.py
            └── test_oauth_oidc.py
```

---

# 9. Domain Layer Rules

The domain is pure Python.

It must not import:

```text
FastAPI
SQLAlchemy
Pydantic
Redis
Celery
PostgreSQL
JWT libraries
HTTP clients
WhatsApp SDK
SMTP libraries
```

The domain contains:

- Aggregates
- Entities
- Value objects
- Domain services
- Domain events
- Domain exceptions

---

# 10. Application Layer Rules

The application layer contains use cases.

Examples:

```text
CreateUser
Login
Logout
RefreshToken
SwitchTenant
AssignRole
GrantPermission
SetupMFA
VerifyMFA
RequestWhatsAppOTP
VerifyWhatsAppOTP
```

Application code may depend on domain abstractions.

It must not directly depend on infrastructure implementations.

---

# 11. Infrastructure Layer

Infrastructure contains implementations:

```text
SQLAlchemy
PostgreSQL
FastAPI
Redis
Celery
JWT
OAuth
OIDC
WebAuthn
SMTP
SMS provider
WhatsApp provider
Message broker
```

Infrastructure implements application ports.

---

# 12. Persistence Schema

Use the previously generated production schema as the persistence baseline.

The major table groups are:

## Identity

```text
identity_iam_user
identity_iam_user_profile
```

## Organization Access

```text
identity_iam_tenant_assignment
identity_iam_company_assignment
identity_iam_branch_assignment
identity_iam_department_assignment
identity_iam_user_category
identity_iam_user_category_assignment
```

## Password / Recovery

```text
identity_iam_password_history
identity_iam_password_reset
identity_iam_email_verification
identity_iam_otp
identity_iam_login_attempt
identity_iam_password_policy
```

## MFA

```text
identity_iam_mfa
identity_iam_mfa_recovery_code
identity_iam_mfa_challenge
identity_iam_webauthn_credential
identity_iam_device
```

## Session

```text
identity_iam_session
```

## OAuth / OIDC

```text
identity_iam_oauth_client
identity_iam_oauth_client_redirect_uri
identity_iam_oauth_client_grant
identity_iam_api_scope
identity_iam_oauth_client_scope
identity_iam_oauth_authorization_code
identity_iam_refresh_token
```

## RBAC

```text
identity_iam_resource
identity_iam_permission
identity_iam_role
identity_iam_role_hierarchy
identity_iam_role_permission
identity_iam_user_role
identity_iam_user_permission
```

## Service Accounts

```text
identity_iam_service_account
identity_iam_service_account_permission
```

## SSO

```text
identity_iam_sso_provider
identity_iam_sso_identity
```

## Security

```text
identity_iam_security_policy
identity_iam_time_based_policy
identity_iam_audit_event
identity_iam_security_event
identity_iam_tenant_setting
```

---

# 13. Domain Aggregates

Do not create one aggregate for every database table.

Recommended aggregates:

```text
UserAggregate
SessionAggregate
RoleAggregate
OAuthClientAggregate
ServiceAccountAggregate
SecurityPolicyAggregate
```

Supporting persistence tables can remain entities or persistence-only records.

---

# 14. User Aggregate

The user aggregate is responsible for rules such as:

```text
activate
deactivate
lock
unlock
change password
verify email
enable authentication method
```

It must not perform database queries.

Example conceptual API:

```python
user.change_password(new_password)
user.verify_email()
user.lock(reason)
user.unlock()
user.deactivate()
```

---

# 15. Authentication Flow

Login should follow:

```text
POST /auth/login
        ↓
Validate credentials
        ↓
Load user
        ↓
Check status
        ↓
Check lock
        ↓
Check password policy
        ↓
Check tenant eligibility
        ↓
Check MFA policy
        ↓
MFA if required
        ↓
Create session
        ↓
Create refresh-token family
        ↓
Issue access token
        ↓
Return authentication response
```

Never put this logic in the FastAPI router.

---

# 16. Password Security

Use Argon2id.

Never store:

```text
plain password
```

Never log:

```text
password
OTP
refresh token
client secret
MFA secret
```

Store only hashes where possible.

Secrets that must be recovered, such as TOTP secrets or provider credentials, must be encrypted using a proper encryption/key-management abstraction.

Password history must prevent reuse according to the tenant password policy.

---

# 17. Password Reset

Flow:

```text
POST /auth/password/reset/request
        ↓
Generate random token
        ↓
Store token hash
        ↓
Send reset message
        ↓
User submits token
        ↓
Verify hash
        ↓
Check expiry
        ↓
Change password
        ↓
Invalidate outstanding sessions/tokens as configured
        ↓
Mark reset token used
        ↓
Audit event
```

Use one-time random tokens.

Never put the raw reset token in the database.

---

# 18. Email Verification

Flow:

```text
Register user
   ↓
Create email verification token
   ↓
Store hash
   ↓
Send email
   ↓
User verifies
   ↓
verified_at
```

Token must:

- expire
- be single-use
- have attempt/rate limits
- be auditable

---

# 19. OTP System

The generic OTP table supports:

```text
LOGIN
PHONE_VERIFICATION
EMAIL_VERIFICATION
PASSWORD_RESET
MFA
TRANSACTION
```

Store:

```text
otp_hash
purpose
delivery_method
destination_hash
expires_at
attempt_count
max_attempts
used_at
```

Never store the plain OTP.

Use cryptographically secure random generation.

---

# 20. WhatsApp OTP

WhatsApp OTP is an official authentication delivery channel in the Identity Platform.

Architecture:

```text
Identity
    ↓
WhatsAppGateway Port
    ↓
WhatsApp Provider Adapter
    ↓
WhatsApp Business API/provider
```

Do not call a WhatsApp SDK directly from the application/domain layer.

Recommended files:

```text
application/commands/whatsapp/
    request_whatsapp_otp.py
    verify_whatsapp_otp.py
    resend_whatsapp_otp.py

application/ports/gateways/
    whatsapp_gateway.py

infrastructure/gateways/whatsapp/
    whatsapp_gateway.py
```

The gateway should expose an abstraction similar to:

```python
class WhatsAppGateway(Protocol):
    async def send_otp(
        self,
        phone_number: str,
        otp: str,
        purpose: str,
    ) -> None:
        ...
```

Provider configuration must come from environment/secret management.

Never hard-code:

```text
access token
API key
phone number ID
business account ID
```

---

# 21. WhatsApp OTP Flow

Request:

```http
POST /api/v1/auth/whatsapp-otp/request
```

Flow:

```text
Phone Number
      ↓
Normalize E.164
      ↓
Find user / verify allowed flow
      ↓
Rate-limit
      ↓
Generate OTP
      ↓
Hash OTP
      ↓
Store identity_iam_otp
      ↓
WhatsApp Gateway
      ↓
Provider
      ↓
WhatsApp message
```

Verify:

```http
POST /api/v1/auth/whatsapp-otp/verify
```

Flow:

```text
OTP
 ↓
Find active challenge
 ↓
Check expiry
 ↓
Check attempts
 ↓
Hash submitted OTP
 ↓
Constant-time comparison
 ↓
Mark used
 ↓
Continue authentication / verification
```

Security requirements:

- short expiration
- maximum verification attempts
- resend cooldown
- per-user rate limit
- per-phone rate limit
- per-IP rate limit
- anti-enumeration responses
- audit events
- abuse monitoring
- no OTP logging

---

# 22. WhatsApp Provider Independence

Do not couple the Identity domain to one vendor.

Use:

```text
WhatsAppGateway
```

with adapters such as:

```text
Meta WhatsApp Cloud API
Twilio
Other approved provider
```

The provider can be changed without changing the domain or application use cases.

---

# 23. MFA

Support:

```text
TOTP
SMS OTP
Email OTP
WhatsApp OTP
WebAuthn/passkey
Recovery codes
```

MFA should be policy-driven.

Example:

```text
Tenant security policy
    ↓
Admin requires MFA
    ↓
Admin login
    ↓
Password valid
    ↓
MFA challenge
```

---

# 24. TOTP

TOTP secret must be encrypted.

Never store a raw secret in plaintext.

Flow:

```text
Setup MFA
 ↓
Generate secret
 ↓
Encrypt secret
 ↓
Display QR/otpauth URI
 ↓
User verifies code
 ↓
Enable MFA
```

---

# 25. Recovery Codes

Recovery codes must be:

```text
random
high entropy
hashed
single-use
```

Never store recovery codes in plaintext.

When a recovery code is used:

```text
used_at = current timestamp
```

---

# 26. WebAuthn / Passkeys

Support:

```text
registration
authentication
credential revocation
sign counter
device naming
backup state
transport metadata
```

The private key never belongs in your database.

Store the credential public key and required WebAuthn metadata.

---

# 27. Device Management

A device can be associated with a user.

Support:

```text
trusted device
device name
browser
OS
device type
last used
revocation
```

This allows:

```text
My Devices
Active Sessions
Trust Device
Revoke Device
```

---

# 28. Session Management

Each successful login creates a session.

Session should contain:

```text
user_id
tenant_id
company_id
branch_id
client_id
device_id
authentication_method
mfa_verified
last_activity_at
expires_at
logout_at
revoked_at
IP
user agent
status
```

Support:

```text
logout current session
logout all sessions
revoke one session
admin force logout
session expiration
idle timeout
absolute timeout
maximum sessions
```

---

# 29. Access Tokens

Use short-lived access tokens.

Recommended baseline:

```text
Access Token: 5–15 minutes
Refresh Token: configurable, normally days/weeks
```

Access tokens should be signed with asymmetric keys in production.

Recommended:

```text
RS256
or
ES256
```

Key rotation must be supported.

---

# 30. Refresh Token Rotation

Use token families.

Example:

```text
RT-1
 ↓
RT-2
 ↓
RT-3
 ↓
RT-4
```

If an already-used refresh token is reused:

```text
REUSE DETECTED
```

Revoke the entire token family.

Required fields:

```text
family_id
parent_token_id
replaced_by_token_id
used_at
revoked_at
expires_at
```

---

# 31. JWT Validation

Every API request must validate:

```text
signature
iss
aud
exp
nbf
iat
jti
```

where applicable.

Do not trust unsigned or incorrectly signed tokens.

Do not accept arbitrary algorithms from the token header.

---

# 32. Authorization

Authorization is separate from authentication.

Authentication:

```text
Who are you?
```

Authorization:

```text
What can you do?
```

The authorization service resolves:

```text
User
 ↓
Tenant context
 ↓
Roles
 ↓
Permissions
 ↓
Policy
 ↓
ALLOW / DENY
```

---

# 33. Permission Naming

Use explicit permission codes.

Example:

```text
finance.invoice.read
finance.invoice.create
finance.invoice.update
finance.invoice.delete
finance.invoice.approve
finance.invoice.export
```

Avoid vague permissions such as:

```text
invoice
admin
manager
```

---

# 34. Role Model

Example:

```text
Finance Manager
```

can have:

```text
finance.invoice.read
finance.invoice.create
finance.invoice.update
finance.invoice.approve
```

Roles can be scoped.

Example:

```text
Tenant-level
Company-level
Branch-level
```

A tenant-level role may apply to all companies and branches within that tenant.

---

# 35. ALLOW / DENY

Authorization must have deterministic precedence.

Recommended:

```text
Explicit DENY
    overrides
ALLOW
```

Never allow a user to accidentally receive both conflicting effects without deterministic resolution.

---

# 36. Role Hierarchy

If role inheritance is supported:

```text
Admin
 ↓
Manager
 ↓
Employee
```

Keep hierarchy semantics explicit.

Prevent:

```text
role A → role B → role A
```

cycles.

---

# 37. User Direct Permissions

Direct permissions should be supported for exceptional cases.

Example:

```text
User
  + finance.invoice.export
```

Direct permissions must also support:

```text
scope
effect
grant timestamp
expiration
granting user
```

---

# 38. Authorization Algorithm

For an operation:

```text
finance.invoice.approve
```

evaluate:

```text
1. Token valid?
2. User active?
3. Session valid?
4. Tenant assignment active?
5. Company assignment active?
6. Branch assignment active?
7. OAuth scope sufficient?
8. Role permission exists?
9. Direct permission exists?
10. Explicit DENY?
11. Permission expired?
12. Security policy satisfied?
13. Time policy satisfied?
14. Device policy satisfied?
15. MFA policy satisfied?
```

Only then:

```text
ALLOW
```

---

# 39. Queries and Tenant Isolation

CQRS queries may directly use SQLAlchemy Core/lightweight SQL.

But they MUST still apply tenant context.

Never do:

```sql
SELECT *
FROM fin.invoice
WHERE id = :id;
```

without authorization/context constraints.

Use:

```sql
SELECT *
FROM fin.invoice
WHERE id = :id
  AND tenant_id = :tenant_id
  AND company_id = :company_id
  AND branch_id = :branch_id;
```

depending on the resource scope.

The context comes from the verified access token.

---

# 40. OAuth 2.1 / OIDC

The Identity Platform should eventually act as an Identity Provider.

Support:

```text
/oauth/authorize
/oauth/token
/oauth/revoke
/oauth/introspect
/oauth/userinfo

/.well-known/openid-configuration
/.well-known/jwks.json
```

OAuth clients should support:

```text
client_id
client_secret_hash
client type
redirect URIs
grant types
scopes
PKCE
token authentication method
token lifetimes
active state
```

---

# 41. OAuth Authorization Code + PKCE

For public clients:

```text
Frontend
  ↓
Authorization Request
  ↓
Login
  ↓
MFA
  ↓
Consent
  ↓
Authorization Code
  ↓
Token Endpoint
  ↓
PKCE verification
  ↓
Access + Refresh Token
```

Never put an authorization code directly into an access token.

Authorization codes must:

- be short-lived
- be single-use
- be bound to the client
- be bound to redirect URI
- support PKCE
- be stored hashed

---

# 42. OAuth Client Scopes

OAuth scopes answer:

```text
What may this application request?
```

ERP permissions answer:

```text
What may this user do?
```

Both should be evaluated.

Example:

```text
OAuth:
finance

User permission:
finance.invoice.approve
```

The final operation requires both appropriate application scope and user authorization.

---

# 43. OIDC

Implement:

```text
openid
profile
email
```

and required discovery metadata.

OIDC should expose claims derived from the authenticated identity.

Do not expose sensitive internal data unnecessarily.

---

# 44. JWKS and Key Rotation

Production Identity requires signing key management.

Implement:

```text
active signing key
previous signing keys
key ID (kid)
JWKS endpoint
rotation process
```

During rotation:

```text
Old key → verify only
New key → sign + verify
```

Do not immediately remove old keys if valid tokens still exist.

---

# 45. SSO / Federation

Tenant-level SSO providers belong in IAM.

Support:

```text
OIDC
SAML
```

Provider records contain configuration and claim mapping.

External identity mapping:

```text
provider
 +
subject
 →
IAM user
```

Never identify an external account only by email if the provider's stable subject is available.

---

# 46. Service Accounts

Background systems should not use human credentials.

Use:

```text
Service Account
    ↓
Client Credentials
    ↓
Access Token
    ↓
API
```

Example:

```text
finance-worker
    ↓
finance.invoice.read
finance.invoice.post
```

Service accounts should have:

```text
owner
tenant
permissions
active status
credential rotation
audit
```

---

# 47. Organization Gateway

IAM must not own ORG business logic.

Use an application port:

```python
class OrganizationGateway(Protocol):
    async def tenant_exists(self, tenant_id: UUID) -> bool:
        ...

    async def company_belongs_to_tenant(
        self,
        company_id: UUID,
        tenant_id: UUID,
    ) -> bool:
        ...

    async def branch_belongs_to_company(
        self,
        branch_id: UUID,
        company_id: UUID,
    ) -> bool:
        ...
```

For a modular monolith this can be a direct adapter.

For future microservices it can become an HTTP/event adapter.

---

# 48. Tenant/Company/Branch Integrity

When assigning:

```text
User → Company
```

verify:

```text
Company belongs to Tenant
```

When assigning:

```text
User → Branch
```

verify:

```text
Branch belongs to Company
Company belongs to Tenant
User belongs to Tenant
```

Never trust IDs supplied by clients.

---

# 49. Event-Driven Architecture

IAM should publish domain/integration events.

Recommended events:

```text
UserCreatedV1
UserUpdatedV1
UserDeletedV1

UserLoggedInV1
UserLoggedOutV1

SessionCreatedV1
SessionRevokedV1

ContextSwitchedV1

TenantAssignmentCreatedV1
TenantAssignmentRevokedV1

CompanyAssignmentCreatedV1
CompanyAssignmentRevokedV1

BranchAssignmentCreatedV1
BranchAssignmentRevokedV1

RoleAssignedV1
RoleRevokedV1

PermissionGrantedV1
PermissionRevokedV1

PasswordChangedV1
PasswordResetV1

MFAEnabledV1
MFADisabledV1

SecurityEventDetectedV1
```

Version events.

Do not silently change event contracts.

---

# 50. Transactional Outbox

Never do:

```python
await db.commit()
await broker.publish(event)
```

as two unrelated operations.

Use:

```text
Database Transaction
│
├── business state
│
└── outbox event
        ↓
      COMMIT
        ↓
Outbox Dispatcher
        ↓
Message Broker
```

This prevents lost events.

---

# 51. Idempotent Event Processing

Consumers must be idempotent.

If:

```text
UserCreatedV1
```

is delivered twice, the consumer must not create two records.

Use:

```text
event_id
consumer_id
processed_at
```

or equivalent deduplication.

---

# 52. Redis

Use Redis for:

```text
rate limiting
short-lived OTP state if appropriate
caching
distributed locks where required
Celery broker/result support
temporary authorization state
```

Do not use Redis as the permanent source of truth for users, roles or sessions.

PostgreSQL remains authoritative.

---

# 53. Rate Limiting

Rate-limit at minimum:

```text
login
password reset
OTP request
OTP verification
WhatsApp OTP request
WhatsApp OTP verification
MFA verification
OAuth authorization
token endpoint
```

Use layered limits:

```text
IP
user
phone
client
tenant
```

Return generic responses where necessary to avoid user enumeration.

---

# 54. Security Audit

Record security-sensitive operations:

```text
login success
login failure
logout
password change
password reset
MFA enable
MFA disable
MFA failure
new device
device revoked
session revoked
role assigned
role revoked
permission changed
tenant access changed
context switched
OAuth client created
SSO configuration changed
service account created
token reuse detected
```

Audit logs should be append-oriented and protected from ordinary user modification.

---

# 55. Security Events

Security events are different from ordinary audit events.

Examples:

```text
BRUTE_FORCE_DETECTED
REFRESH_TOKEN_REUSE
IMPOSSIBLE_TRAVEL
SUSPICIOUS_IP
MFA_BRUTE_FORCE
ACCOUNT_TAKEOVER_SUSPECTED
UNUSUAL_DEVICE
```

Include:

```text
severity
user
tenant
IP
timestamp
details
resolution
```

---

# 56. Secrets

Never commit:

```text
JWT private key
JWT secret
database password
Redis password
WhatsApp token
SMTP password
OAuth client secret
SSO secret
encryption key
```

Use:

```text
.env
secret manager
container secrets
production key management
```

Do not put secrets in Git.

---

# 57. Configuration

Example:

```env
ENVIRONMENT=development

DATABASE_URL=postgresql+asyncpg://USER:PASSWORD@HOST:5432/jesloterp

REDIS_URL=redis://localhost:6379/0

JWT_ISSUER=https://idp.jesloterp.example
JWT_AUDIENCE=jeslot-api

JWT_PRIVATE_KEY_PATH=...
JWT_PUBLIC_KEY_PATH=...

WHATSAPP_PROVIDER=...
WHATSAPP_ACCESS_TOKEN=...
WHATSAPP_PHONE_NUMBER_ID=...

SMTP_HOST=...
SMTP_PORT=...
SMTP_USERNAME=...
SMTP_PASSWORD=...
```

Use a typed Pydantic settings object.

---

# 58. API Structure

Recommended routers:

```text
/auth
/users
/roles
/permissions
/contexts
/sessions
/mfa
/oauth
/oidc
/sso
/service-accounts
```

Authentication:

```text
POST /auth/login
POST /auth/logout
POST /auth/refresh
GET  /auth/me
```

Password:

```text
POST /auth/password/change
POST /auth/password/reset/request
POST /auth/password/reset/confirm
```

Email:

```text
POST /auth/email/verify/request
POST /auth/email/verify
```

OTP:

```text
POST /auth/otp/request
POST /auth/otp/verify
POST /auth/otp/resend
```

WhatsApp:

```text
POST /auth/whatsapp-otp/request
POST /auth/whatsapp-otp/verify
POST /auth/whatsapp-otp/resend
```

Context:

```text
GET  /auth/contexts
POST /auth/context/switch
```

Sessions:

```text
GET    /auth/sessions
DELETE /auth/sessions/{session_id}
DELETE /auth/sessions
```

MFA:

```text
POST /auth/mfa/setup
POST /auth/mfa/verify
POST /auth/mfa/disable
POST /auth/mfa/recovery-codes/regenerate
```

---

# 59. Router Rule

FastAPI routers must be thin.

Bad:

```python
@router.post("/login")
async def login(...):
    query_database()
    verify_password()
    generate_jwt()
    create_session()
    publish_event()
```

Good:

```python
@router.post("/login")
async def login(
    request: LoginRequest,
    handler: LoginHandler = Depends(get_login_handler),
):
    command = LoginCommand(
        username=request.username,
        password=request.password,
    )

    return await handler.handle(command)
```

The handler orchestrates the use case.

---

# 60. CQRS

Commands:

```text
write state
```

Queries:

```text
read state
```

Commands use:

```text
Aggregate
Repository
Unit of Work
Domain Events
Outbox
```

Queries can use:

```text
SQLAlchemy Core
optimized SQL
projection tables
Pydantic response DTO
```

Queries do not need to load full domain aggregates.

---

# 61. Repository Rules

Repositories are application ports.

Example:

```python
class UserRepository(Protocol):

    async def get_by_id(
        self,
        user_id: UUID,
    ) -> UserAggregate | None:
        ...

    async def get_by_email(
        self,
        email: str,
    ) -> UserAggregate | None:
        ...

    async def save(
        self,
        user: UserAggregate,
    ) -> None:
        ...
```

Infrastructure implements the interface.

---

# 62. Mapper Rules

ORM ↔ Domain translation belongs in:

```text
infrastructure/persistence/mappers/
```

Example:

```text
SQLAlchemy UserModel
       ↓
UserMapper
       ↓
UserAggregate
```

Never expose SQLAlchemy ORM models to the domain.

---

# 63. Unit of Work

Commands should use a transaction boundary.

Example:

```text
Command
 ↓
UnitOfWork
 ├── repository changes
 ├── aggregate events
 └── outbox records
 ↓
COMMIT
```

Rollback everything on failure.

---

# 64. Domain Events vs Integration Events

Domain event:

```text
UserLoggedIn
```

is an internal domain concept.

Integration event:

```text
UserLoggedInV1
```

is an externally published contract.

Map/version them explicitly.

---

# 65. Error Handling

Use structured application/domain exceptions.

Examples:

```text
UserNotFound
InvalidCredentials
AccountLocked
AccountInactive
MFARequired
InvalidMFA
OTPExpired
OTPAlreadyUsed
InvalidContext
TenantAccessDenied
PermissionDenied
SessionExpired
RefreshTokenReuseDetected
InvalidOAuthClient
InvalidRedirectURI
PKCEVerificationFailed
```

FastAPI maps these to proper HTTP responses.

Do not leak internal database errors.

---

# 66. Authentication Error Security

Do not reveal whether:

```text
email exists
phone exists
username exists
```

where doing so could enable account enumeration.

Use generic responses for public recovery/login endpoints where appropriate.

---

# 67. Database Migration

Use Alembic.

Never manually edit production schema outside migration management.

Workflow:

```bash
uv run alembic revision --autogenerate -m "add identity feature"
```

Then manually review:

```text
alembic/versions/
```

Verify:

- schema name
- indexes
- unique constraints
- foreign keys
- nullable behavior
- delete behavior
- data migration
- security implications

Then:

```bash
uv run alembic upgrade head
```

---

# 68. Schema Naming

Use:

```text
iam
```

as the PostgreSQL schema for the Identity Platform.

Examples:

```text
iam.identity_iam_user
iam.identity_iam_role
iam.identity_iam_permission
```

Other modules:

```text
org
fin
sales
purchase
inventory
```

The exact schema prefixes must be consistent in models and migrations.

---

# 69. Database Indexing

Index:

```text
user email_normalized
user username
tenant assignments
company assignments
branch assignments
session user
session status
session expiry
refresh token family
refresh token expiry
OAuth client ID
OAuth authorization code expiry
role tenant
role permission
user role
user permission
audit tenant + timestamp
security event tenant + timestamp
```

Review indexes based on actual query plans as the system grows.

---

# 70. Data Retention

Define retention policies for:

```text
login attempts
OTP records
audit events
security events
expired sessions
expired tokens
password reset records
email verification records
```

Do not allow security tables to grow indefinitely without lifecycle management.

Use scheduled cleanup jobs.

---

# 71. Background Workers

Celery/background workers handle:

```text
email delivery
WhatsApp delivery where asynchronous
SMS delivery
outbox publishing
audit archival
token cleanup
OTP cleanup
session cleanup
security analysis
notification retries
```

Do not block API requests for long-running external operations unless required.

---

# 72. Retry Rules

External messaging should use retries.

Use:

```text
exponential backoff
maximum retry count
dead-letter handling
idempotency
```

Do not endlessly retry failed WhatsApp/email/SMS operations.

---

# 73. WhatsApp Operational Rules

For WhatsApp OTP specifically:

```text
Provider failure
    ↓
Retry according to policy
    ↓
Do not create duplicate OTPs unnecessarily
```

A resend should either:

```text
invalidate previous challenge
```

or follow an explicit challenge versioning strategy.

Only the latest valid challenge should normally be accepted.

---

# 74. Organization Integration

IAM should communicate with ORG through:

```text
OrganizationGateway
```

ORG is authoritative for:

```text
tenant existence
company existence
branch existence
department existence
company → tenant relationship
branch → company relationship
```

IAM is authoritative for:

```text
user → tenant access
user → company access
user → branch access
user → department access
```

---

# 75. Finance Integration

Finance must not directly manipulate IAM database tables.

Finance should depend on:

```text
AuthorizationPort
```

Example:

```python
await authorization.authorize(
    context,
    "finance.invoice.approve",
)
```

Then Finance performs its own business logic.

---

# 76. Business Module Dependency Rule

Correct:

```text
Finance
  ↓
AuthorizationPort
  ↓
IAM adapter
```

Incorrect:

```text
Finance
  ↓
identity_iam_user SQLAlchemy model
```

Business modules must not become coupled to IAM persistence.

---

# 77. Example Finance Flow

```text
POST /finance/invoices/{id}/approve
        ↓
FastAPI
        ↓
RequestContext
        ↓
SubmitInvoiceCommand
        ↓
SubmitInvoiceHandler
        ↓
AuthorizationPort
        ↓
finance.invoice.approve
        ↓
InvoiceRepository
        ↓
InvoiceAggregate
        ↓
Business rule
        ↓
Outbox
        ↓
Commit
```

---

# 78. Platform Admin vs Tenant Admin

Do not treat all admins as the same.

Use separate concepts:

```text
Platform Administrator
```

for the entire JeslotERP platform.

And:

```text
Tenant Administrator
```

for one tenant.

A platform administrator may have platform-wide authority.

A tenant administrator must remain tenant-scoped.

Do not make `is_platform_admin` equivalent to every tenant permission.

---

# 79. Super Admin Safety

Platform administration must be extremely restricted.

Recommended:

```text
MFA required
strong password
short sessions
audit every action
IP policy if appropriate
separate administrative client
re-authentication for sensitive actions
```

---

# 80. Sensitive Operations

Require elevated authentication/re-authentication for operations such as:

```text
change password
disable MFA
regenerate recovery codes
create service account
change SSO configuration
change OAuth credentials
assign platform admin
change high-risk permissions
```

---

# 81. Test Strategy

## Unit tests

Test:

```text
UserAggregate
RoleAggregate
PermissionResolver
TenantContextService
PasswordPolicyService
AuthorizationService
```

without PostgreSQL.

## Integration tests

Test:

```text
PostgreSQL
Redis
repositories
JWT
OAuth
MFA
WhatsApp gateway
outbox
```

## E2E tests

Test complete flows:

```text
registration
login
MFA
logout
refresh
context switch
role authorization
tenant isolation
WhatsApp OTP
OAuth authorization code
PKCE
SSO
```

---

# 82. Critical Security Test Cases

You must test:

```text
user from Tenant A cannot access Tenant B
company from Tenant A cannot be assigned to Tenant B
branch from Company A cannot be used under Company B
expired token rejected
wrong audience rejected
wrong issuer rejected
revoked session rejected
revoked refresh token rejected
refresh token reuse revokes family
expired OTP rejected
used OTP rejected
wrong OTP limited
MFA bypass impossible
DENY overrides ALLOW
expired permission rejected
inactive user rejected
inactive tenant assignment rejected
OAuth redirect URI mismatch rejected
PKCE mismatch rejected
unauthorized service account rejected
```

---

# 83. CI Quality Gates

Run:

```bash
uv run mypy .
uv run pytest tests/ -v
```

Also add:

```text
ruff
format checking
import boundary checks
migration checks
security scanning
```

The CI pipeline should fail if domain/application imports infrastructure dependencies.

---

# 84. Local Environment

Use:

```bash
uv sync
```

Infrastructure:

```bash
docker compose up -d db redis minio
```

Database:

```bash
uv run alembic upgrade head
```

Seed:

```bash
uv run python scripts/seed_master_tenant.py
```

Run API:

```bash
uv run uvicorn apps.api.main:app --reload --port 8000
```

Run worker:

```bash
uv run celery -A apps.worker.celery_app worker --loglevel=info
```

---

# 85. Environment Variables

Never use production credentials locally.

Example:

```env
ENVIRONMENT=development

DATABASE_URL=postgresql+asyncpg://USER:PASSWORD@HOST:5432/jesloterp

REDIS_URL=redis://localhost:6379/0

JWT_ISSUER=http://localhost:8000
JWT_AUDIENCE=jeslot-api

WHATSAPP_PROVIDER=development
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=

SMTP_HOST=
SMTP_PORT=587
SMTP_USERNAME=
SMTP_PASSWORD=
```

---

# 86. Production Deployment

Production should have:

```text
Load Balancer
      ↓
API
      ↓
IAM Application
      ↓
PostgreSQL
      ↓
Redis

Worker
      ↓
Outbox
      ↓
Broker
      ↓
External providers
```

Use:

```text
TLS
secret manager
key rotation
database backups
monitoring
centralized logs
metrics
tracing
health checks
```

---

# 87. Health Checks

Expose separate checks:

```text
/live
/ready
```

Liveness:

```text
process is alive
```

Readiness:

```text
database available
required dependencies available
```

Do not expose secrets or sensitive system details through health endpoints.

---

# 88. Observability

Every request should have:

```text
request_id
trace_id
user_id where available
tenant_id where available
```

Do not log:

```text
password
OTP
access token
refresh token
session token
client secret
MFA secret
```

---

# 89. Logging

Use structured logging.

Example:

```json
{
  "event": "user.login.success",
  "request_id": "...",
  "user_id": "...",
  "tenant_id": "...",
  "ip": "...",
  "timestamp": "..."
}
```

Sensitive values must be redacted.

---

# 90. Cache Strategy

Cache only safe/rebuildable information.

Potential cache:

```text
permission resolution
role hierarchy
public signing keys
OIDC discovery
tenant security policy
```

Always define invalidation when:

```text
role changed
permission changed
user permission changed
tenant policy changed
```

Security correctness takes priority over cache performance.

---

# 91. Authorization Cache

If permissions are cached, use a version/invalidation strategy.

Example:

```text
tenant permission version = 42
```

When role/permission changes:

```text
version = 43
```

Old cache entries become invalid.

Never allow stale authorization to remain indefinitely.

---

# 92. Token Revocation Strategy

JWTs are stateless, but sessions are stateful.

Recommended:

```text
Access token short lifetime
+
session validation for high-security operations
+
refresh token revocation
```

For immediate global revocation, maintain a server-side revocation/version mechanism.

---

# 93. Database as Source of Truth

Authoritative storage:

```text
PostgreSQL
```

Temporary/distributed infrastructure:

```text
Redis
```

Events:

```text
Message broker
```

Never make a cache or event stream the source of truth for IAM state.

---

# 94. What NOT to Do

Never:

```text
put tenant_id directly on global user
trust X-Tenant-ID for authorization
store raw passwords
store raw OTPs
store raw reset tokens
store raw refresh tokens
log authentication secrets
put SQLAlchemy models in domain
put business logic in FastAPI routers
publish events before DB commit
let Finance directly query IAM tables
put all permissions in JWT
use human credentials for workers
skip tenant isolation in queries
allow arbitrary OAuth redirect URIs
disable PKCE for public clients
```

---

# 95. Recommended Implementation Order

Do not implement all functionality simultaneously.

## Phase 1

```text
User
Profile
Tenant assignment
Company assignment
Branch assignment
Department assignment
```

## Phase 2

```text
Password
Login
Logout
Session
Password reset
Email verification
Login attempts
```

## Phase 3

```text
Resource
Permission
Role
Role permission
User role
User permission
Authorization service
```

## Phase 4

```text
RequestContext
Tenant context
Company context
Branch context
Context switch
```

## Phase 5

```text
JWT
Refresh tokens
Rotation
Reuse detection
JWKS
Key rotation
```

## Phase 6

```text
MFA
TOTP
Recovery codes
MFA challenges
WebAuthn
```

## Phase 7

```text
WhatsApp OTP
SMS
Email OTP
Rate limiting
Provider adapters
```

## Phase 8

```text
OAuth
OIDC
PKCE
Authorization code
Client scopes
Discovery
UserInfo
```

## Phase 9

```text
SSO
OIDC federation
SAML
External identity mapping
```

## Phase 10

```text
Service accounts
Client credentials
Security policies
Audit
Security events
Outbox
Advanced observability
```

---

# 96. Final System Architecture

The final architecture should be:

```text
                         FRONTEND
                            │
                            │ Bearer Access Token
                            ▼
                    ┌───────────────┐
                    │ API Gateway   │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Identity IAM  │
                    │               │
                    │ Authentication│
                    │ Authorization │
                    │ OAuth / OIDC  │
                    │ MFA / Passkey │
                    │ SSO           │
                    └───────┬───────┘
                            │
                   RequestContext
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Finance         Sales        Purchase
             │              │              │
             ▼              ▼              ▼
          Domain         Domain         Domain
             │              │              │
             ▼              ▼              ▼
        Repository      Repository      Repository
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                       PostgreSQL
                            │
                          Outbox
                            │
                            ▼
                       Message Broker
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          Notification    Audit        Workers
              │
       ┌──────┴──────┐
       ▼             ▼
     Email        WhatsApp
```

---

# 97. Final Ownership Model

| Responsibility | Owner |
|---|---|
| User identity | IAM |
| User profile | IAM |
| Password | IAM |
| Login | IAM |
| OTP | IAM |
| WhatsApp OTP | IAM |
| MFA | IAM |
| Passkeys | IAM |
| Sessions | IAM |
| Access tokens | IAM |
| Refresh tokens | IAM |
| OAuth/OIDC | IAM |
| SSO | IAM |
| Roles | IAM |
| Permissions | IAM |
| Authorization | IAM |
| Tenant master | ORG |
| Company master | ORG |
| Branch master | ORG |
| Department master | ORG |
| Finance data | Finance |
| Sales data | Sales |
| Purchase data | Purchase |
| Inventory data | Inventory |
| Business rules | Domain |
| Database implementation | Infrastructure |
| HTTP implementation | Infrastructure |
| External provider integration | Infrastructure |
| Events | Outbox/Messaging |

---

# 98. Golden Rules

1. **User is global.**
2. **Tenant/company/branch belong to ORG.**
3. **User access to tenant/company/branch belongs to IAM.**
4. **Never trust tenant IDs supplied by the frontend.**
5. **Context comes from a verified token.**
6. **Context switching creates a new context-bound token.**
7. **Authentication and authorization are separate.**
8. **Roles and permissions belong to IAM.**
9. **Business modules must not query IAM persistence directly.**
10. **Domain must remain framework-independent.**
11. **Commands use aggregates and repositories.**
12. **Queries can use optimized SQL/read models.**
13. **Use Unit of Work for command transactions.**
14. **Use the transactional outbox for events.**
15. **Use idempotent consumers.**
16. **Use Argon2id for passwords.**
17. **Never store plaintext authentication secrets.**
18. **Use short-lived access tokens.**
19. **Rotate refresh tokens.**
20. **Detect refresh-token reuse and revoke the family.**
21. **Use asymmetric JWT signing in production.**
22. **Support JWKS and key rotation.**
23. **Use PKCE for authorization code clients.**
24. **Never allow arbitrary OAuth redirect URIs.**
25. **MFA must be policy-driven.**
26. **WhatsApp OTP must be behind a provider gateway.**
27. **Rate-limit authentication and OTP operations.**
28. **Audit security-sensitive actions.**
29. **Test tenant isolation explicitly.**
30. **Treat security correctness as more important than convenience.**

---

# 99. Definition of Done

The Identity Platform is considered production-ready only when:

```text
[ ] Global user identity works
[ ] Multiple tenant memberships work
[ ] Company/branch assignments work
[ ] Context switching works
[ ] Context is included in verified tokens
[ ] Password authentication works
[ ] Password reset works
[ ] Email verification works
[ ] Login rate limiting works
[ ] Session management works
[ ] Logout-all works
[ ] JWT validation works
[ ] Refresh token rotation works
[ ] Refresh token reuse detection works
[ ] RBAC works
[ ] Permission resolution works
[ ] DENY precedence works
[ ] Permission expiry works
[ ] Tenant isolation is tested
[ ] MFA works
[ ] Recovery codes work
[ ] WebAuthn works
[ ] WhatsApp OTP works
[ ] OTP rate limiting works
[ ] OAuth authorization code works
[ ] PKCE works
[ ] OIDC discovery works
[ ] JWKS works
[ ] Key rotation works
[ ] Service accounts work
[ ] SSO works
[ ] Audit events work
[ ] Security events work
[ ] Outbox works
[ ] Event consumers are idempotent
[ ] PostgreSQL migrations work
[ ] Redis integration works
[ ] Celery workers work
[ ] CI type checking works
[ ] Unit tests pass
[ ] Integration tests pass
[x] TASK-SOR-028: shipped SSO/MFA/WebAuthn threat paths tested (no live IdP; SAML ACS 501)
[ ] E2E tests pass
[ ] Secrets are externalized
[ ] Production TLS is enabled
[ ] Backups are configured
[ ] Monitoring and alerting are configured
```

---

# 100. Final Principle

The Identity Platform is the **security authority**, not the business-data authority.

The final separation is:

```text
                    JESLOTERP
                        │
        ┌───────────────┴────────────────┐
        │                                │
       ORG                             IAM
        │                                │
        │                                ├── Users
        ├── Tenants                      ├── Authentication
        ├── Companies                    ├── MFA
        ├── Branches                     ├── Sessions
        └── Departments                  ├── OAuth/OIDC
                                         ├── SSO
                                         ├── Roles
                                         ├── Permissions
                                         └── Authorization
```

And business modules remain independent:

```text
IAM
 │
 └── Security Port
       │
       ├── Finance
       ├── Sales
       ├── Purchase
       ├── Inventory
       └── Other ERP Modules
```

This allows JeslotERP to start as a modular monolith and later extract IAM into a standalone Identity Provider without redesigning the business domains.
