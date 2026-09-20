# JeslotERP Identity & IAM Platform — Complete API Endpoint Specification

## 1. Purpose

This document defines the production-grade API surface for the JeslotERP Identity & IAM Platform.

IAM owns identity, authentication, authorization, sessions, tokens, MFA, SSO, service accounts, security policies, audit/security events, and user access assignments.

The ORG module remains the owner of organization master data:
- Tenant
- Company
- Branch
- Department

IAM manages which users can access those organization objects.

---

## 2. Base URLs

Public API:

```text
https://idp.jesloterp.example/api/v1
```

OAuth/OIDC:

```text
https://idp.jesloterp.example/oauth
```

OIDC discovery:

```text
https://idp.jesloterp.example/.well-known/openid-configuration
https://idp.jesloterp.example/.well-known/jwks.json
```

Internal service API:

```text
https://idp.jesloterp.example/internal/v1
```

---

## 3. Authentication and Context

Protected APIs use:

```http
Authorization: Bearer <access_token>
```

Do not use `X-Tenant-ID`, `X-Company-ID`, or `X-Branch-ID` as authoritative authorization context.

Recommended JWT claims:

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

Application services should convert this to:

```python
@dataclass(frozen=True)
class RequestContext:
    user_id: UUID
    tenant_id: UUID | None
    company_id: UUID | None
    branch_id: UUID | None
    session_id: UUID | None
    scopes: frozenset[str]
```

When context changes, issue a new context-bound access token.

---

# 4. Health and Platform

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Basic health |
| GET | `/health/live` | Liveness |
| GET | `/health/ready` | Readiness |
| GET | `/version` | Version information |
| GET | `/metrics` | Prometheus metrics |

`/metrics` should normally be internal/protected.

---

# 5. Authentication APIs

## Login

```http
POST /api/v1/auth/login
```

Responsibilities:
- Validate credentials
- Apply account status rules
- Apply security policy
- Apply rate limits
- Detect MFA requirement
- Create session
- Issue tokens

Example request:

```json
{
  "identifier": "user@example.com",
  "password": "********",
  "client_id": "web-client"
}
```

## Logout

```http
POST /api/v1/auth/logout
```

## Logout all sessions

```http
POST /api/v1/auth/logout-all
```

## Current authentication

```http
GET /api/v1/auth/me
GET /api/v1/auth/session
GET /api/v1/auth/sessions
```

---

# 6. Password APIs

```http
POST /api/v1/auth/password/change
POST /api/v1/auth/password/forgot
POST /api/v1/auth/password/reset
POST /api/v1/auth/password/verify
```

Password requirements:
- Argon2id
- Password history
- Configurable password policy
- Rate limiting
- No plaintext storage
- No password logging

Forgot-password responses must not reveal whether an account exists.

---

# 7. Email Verification APIs

```http
POST /api/v1/auth/email/send-verification
POST /api/v1/auth/email/verify
POST /api/v1/auth/email/resend-verification
```

Verification tokens must be:
- Hashed in storage
- Single-use
- Expiring
- Rate limited

---

# 8. Generic OTP APIs

```http
POST /api/v1/otp/request
POST /api/v1/otp/verify
POST /api/v1/otp/resend
```

Supported purposes can include:

```text
LOGIN
REGISTRATION
PASSWORD_RESET
EMAIL_VERIFICATION
PHONE_VERIFICATION
MFA
TRANSACTION
```

OTP must be generated securely, hashed before storage, expire quickly, have attempt limits, and never be logged.

---

# 9. WhatsApp OTP APIs

```http
POST /api/v1/auth/whatsapp/request
POST /api/v1/auth/whatsapp/verify
POST /api/v1/auth/whatsapp/resend
POST /api/v1/auth/whatsapp/change-number
POST /api/v1/auth/whatsapp/verify-number
```

Example request:

```json
{
  "phone_number": "+919876543210",
  "purpose": "LOGIN"
}
```

Example verification:

```json
{
  "challenge_id": "...",
  "otp": "123456"
}
```

Phone numbers must be normalized to E.164.

The WhatsApp implementation should use an application gateway such as:

```python
class WhatsAppGateway(Protocol):
    async def send_otp(
        self,
        phone_number: str,
        otp: str,
    ) -> None:
        ...
```

The application/domain layer must not depend directly on a WhatsApp vendor.

Rate-limit by appropriate combinations of:
- IP
- Phone
- User
- Client
- Tenant

Never log the OTP or provider credentials.

---

# 10. Registration APIs

```http
POST /api/v1/users/register
POST /api/v1/users/register/verify
POST /api/v1/users/register/resend-verification
```

Registration should not automatically grant broad permissions.

Possible flow:

```text
Registration
    ↓
Identity creation
    ↓
Email/phone verification
    ↓
Tenant invitation or tenant creation
    ↓
Initial role assignment
```

---

# 11. User APIs

## CRUD

```http
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/{user_id}
PATCH  /api/v1/users/{user_id}
DELETE /api/v1/users/{user_id}
```

## Lifecycle

```http
POST /api/v1/users/{user_id}/activate
POST /api/v1/users/{user_id}/deactivate
POST /api/v1/users/{user_id}/suspend
POST /api/v1/users/{user_id}/restore
POST /api/v1/users/{user_id}/lock
POST /api/v1/users/{user_id}/unlock
```

## Profile

```http
GET   /api/v1/users/{user_id}/profile
PATCH /api/v1/users/{user_id}/profile
```

---

# 12. Current User APIs

These are heavily used by the frontend.

```http
GET   /api/v1/me
PATCH /api/v1/me

GET   /api/v1/me/profile
PATCH /api/v1/me/profile

GET /api/v1/me/tenants
GET /api/v1/me/companies
GET /api/v1/me/branches
GET /api/v1/me/departments

GET /api/v1/me/roles
GET /api/v1/me/permissions

GET /api/v1/me/sessions
GET /api/v1/me/devices

GET /api/v1/me/mfa
GET /api/v1/me/webauthn
GET /api/v1/me/sso-identities
```

---

# 13. User Security APIs

```http
GET /api/v1/users/{user_id}/security
GET /api/v1/users/{user_id}/security-events
GET /api/v1/users/{user_id}/login-history
GET /api/v1/users/{user_id}/devices
GET /api/v1/users/{user_id}/sessions
```

Administrative actions:

```http
POST /api/v1/users/{user_id}/revoke-sessions
POST /api/v1/users/{user_id}/force-password-reset
POST /api/v1/users/{user_id}/force-mfa
```

---

# 14. Tenant Assignment APIs

Tenant master data is owned by ORG. IAM owns user membership.

```http
GET    /api/v1/users/{user_id}/tenants
POST   /api/v1/users/{user_id}/tenants
DELETE /api/v1/users/{user_id}/tenants/{tenant_id}

POST /api/v1/users/{user_id}/tenants/{tenant_id}/activate
POST /api/v1/users/{user_id}/tenants/{tenant_id}/deactivate
```

The assignment can support:
- Active/inactive state
- Default tenant
- Membership type
- Effective dates

---

# 15. Company Assignment APIs

```http
GET    /api/v1/users/{user_id}/companies
POST   /api/v1/users/{user_id}/companies
DELETE /api/v1/users/{user_id}/companies/{company_id}
```

Before assignment, validate:
- User has tenant access
- Company exists
- Company belongs to the tenant

---

# 16. Branch Assignment APIs

```http
GET    /api/v1/users/{user_id}/branches
POST   /api/v1/users/{user_id}/branches
DELETE /api/v1/users/{user_id}/branches/{branch_id}

GET /api/v1/me/branches
```

Validate:

```text
User
 ↓
Tenant membership
 ↓
Company membership
 ↓
Branch belongs to company
```

---

# 17. Department Assignment APIs

```http
GET    /api/v1/users/{user_id}/departments
POST   /api/v1/users/{user_id}/departments
DELETE /api/v1/users/{user_id}/departments/{department_id}
```

---

# 18. Context APIs

```http
GET  /api/v1/context
GET  /api/v1/context/available
POST /api/v1/context/switch
POST /api/v1/context/validate
```

Switch request:

```json
{
  "tenant_id": "...",
  "company_id": "...",
  "branch_id": "..."
}
```

Switch flow:

```text
Authenticate user
      ↓
Check tenant assignment
      ↓
Check company assignment
      ↓
Check branch assignment
      ↓
Validate ORG hierarchy
      ↓
Resolve roles
      ↓
Resolve permissions
      ↓
Create context
      ↓
Issue new access token
```

Do not mutate an existing JWT.

---

# 19. Role APIs

```http
GET    /api/v1/roles
POST   /api/v1/roles
GET    /api/v1/roles/{role_id}
PATCH  /api/v1/roles/{role_id}
DELETE /api/v1/roles/{role_id}

POST /api/v1/roles/{role_id}/activate
POST /api/v1/roles/{role_id}/deactivate
```

---

# 20. Role Hierarchy APIs

```http
GET    /api/v1/roles/{role_id}/children
POST   /api/v1/roles/{role_id}/children
DELETE /api/v1/roles/{role_id}/children/{child_role_id}
```

Example:

```text
Tenant Admin
    ↓
Company Admin
    ↓
Branch Manager
    ↓
Employee
```

Prevent privilege-escalation cycles.

---

# 21. User Role APIs

```http
GET    /api/v1/users/{user_id}/roles
POST   /api/v1/users/{user_id}/roles
DELETE /api/v1/users/{user_id}/roles/{role_id}
```

Role assignments may be scoped to:
- Tenant
- Company
- Branch
- Department

Example:

```json
{
  "role_id": "...",
  "tenant_id": "...",
  "company_id": "...",
  "branch_id": "..."
}
```

---

# 22. Permission APIs

```http
GET    /api/v1/permissions
POST   /api/v1/permissions
GET    /api/v1/permissions/{permission_id}
PATCH  /api/v1/permissions/{permission_id}
DELETE /api/v1/permissions/{permission_id}

GET /api/v1/permissions/by-code/{permission_code}
```

Recommended naming:

```text
{module}.{resource}.{action}
```

Examples:

```text
finance.invoice.view
finance.invoice.create
finance.invoice.update
finance.invoice.delete
finance.invoice.approve

sales.order.view
sales.order.create
sales.order.approve
```

---

# 23. Role Permission APIs

```http
GET    /api/v1/roles/{role_id}/permissions
POST   /api/v1/roles/{role_id}/permissions
DELETE /api/v1/roles/{role_id}/permissions/{permission_id}
POST   /api/v1/roles/{role_id}/permissions/bulk
```

---

# 24. Direct User Permission APIs

```http
GET    /api/v1/users/{user_id}/permissions
POST   /api/v1/users/{user_id}/permissions
DELETE /api/v1/users/{user_id}/permissions/{permission_id}
POST   /api/v1/users/{user_id}/permissions/bulk
```

Direct permissions should be used carefully; roles should be preferred for normal administration.

---

# 25. Resource APIs

```http
GET    /api/v1/resources
POST   /api/v1/resources
GET    /api/v1/resources/{resource_id}
PATCH  /api/v1/resources/{resource_id}
DELETE /api/v1/resources/{resource_id}
```

Example resources:

```text
finance
sales
inventory
hr
fleet
billing
```

---

# 26. Authorization APIs

Single check:

```http
POST /api/v1/authorization/check
```

Example:

```json
{
  "permission": "finance.invoice.approve",
  "tenant_id": "...",
  "company_id": "...",
  "branch_id": "..."
}
```

Response:

```json
{
  "allowed": true
}
```

Bulk check:

```http
POST /api/v1/authorization/check-bulk
```

Example:

```json
{
  "permissions": [
    "finance.invoice.view",
    "finance.invoice.approve",
    "finance.payment.create"
  ]
}
```

Other modules should normally use an `AuthorizationPort`, not IAM database models.

---

# 27. MFA APIs

```http
GET  /api/v1/me/mfa
POST /api/v1/me/mfa/setup
POST /api/v1/me/mfa/verify
POST /api/v1/me/mfa/enable
POST /api/v1/me/mfa/disable
```

Supported methods can include:

```text
TOTP
SMS
Email OTP
WhatsApp OTP
WebAuthn/passkey
Recovery codes
```

---

# 28. TOTP APIs

```http
POST /api/v1/me/mfa/totp/setup
POST /api/v1/me/mfa/totp/verify
POST /api/v1/me/mfa/totp/enable
POST /api/v1/me/mfa/totp/disable
```

TOTP secrets must be encrypted at rest.

---

# 29. MFA Recovery Code APIs

```http
GET  /api/v1/me/mfa/recovery-codes
POST /api/v1/me/mfa/recovery-codes/regenerate
POST /api/v1/me/mfa/recovery-codes/verify
```

Recovery codes should be hashed.

---

# 30. MFA Challenge APIs

```http
POST /api/v1/mfa/challenge
POST /api/v1/mfa/challenge/{challenge_id}/verify
POST /api/v1/mfa/challenge/{challenge_id}/resend
```

Challenges should have:
- Expiration
- Attempt limits
- Purpose
- Method
- User
- Session
- Status

---

# 31. WebAuthn / Passkey APIs

Registration:

```http
POST /api/v1/webauthn/register/options
POST /api/v1/webauthn/register/verify
```

Authentication:

```http
POST /api/v1/webauthn/login/options
POST /api/v1/webauthn/login/verify
```

Credential management:

```http
GET    /api/v1/me/webauthn
DELETE /api/v1/me/webauthn/{credential_id}
```

---

# 32. Device APIs

```http
GET    /api/v1/me/devices
GET    /api/v1/me/devices/{device_id}
PATCH  /api/v1/me/devices/{device_id}
DELETE /api/v1/me/devices/{device_id}

POST /api/v1/me/devices/{device_id}/trust
POST /api/v1/me/devices/{device_id}/untrust
```

Store only necessary device metadata.

---

# 33. Session APIs

Current user:

```http
GET    /api/v1/me/sessions
GET    /api/v1/me/sessions/{session_id}
POST   /api/v1/me/sessions/{session_id}/revoke
DELETE /api/v1/me/sessions/{session_id}
POST   /api/v1/me/sessions/revoke-all
```

Administrative:

```http
GET  /api/v1/users/{user_id}/sessions
POST /api/v1/users/{user_id}/sessions/revoke-all
```

Support:
- Idle timeout
- Absolute timeout
- Maximum sessions
- Session revocation
- Optional device binding

---

# 34. OAuth 2.0 APIs

Standard endpoints:

```http
GET  /oauth/authorize
POST /oauth/token
POST /oauth/revoke
POST /oauth/introspect
GET  /oauth/userinfo
```

Recommended flows:
- Authorization Code + PKCE
- Refresh Token
- Client Credentials

Avoid legacy implicit flow.

---

# 35. OpenID Connect Discovery

```http
GET /.well-known/openid-configuration
GET /.well-known/jwks.json
```

Discovery should describe:
- Authorization endpoint
- Token endpoint
- UserInfo endpoint
- JWKS endpoint
- Supported scopes
- Supported grant types
- Response types
- PKCE methods
- Signing algorithms

---

# 36. OAuth Client APIs

```http
GET    /api/v1/oauth/clients
POST   /api/v1/oauth/clients
GET    /api/v1/oauth/clients/{client_id}
PATCH  /api/v1/oauth/clients/{client_id}
DELETE /api/v1/oauth/clients/{client_id}

POST /api/v1/oauth/clients/{client_id}/activate
POST /api/v1/oauth/clients/{client_id}/deactivate

POST /api/v1/oauth/clients/{client_id}/secret/rotate
POST /api/v1/oauth/clients/{client_id}/secret/revoke
```

Never expose an existing client secret again after creation.

---

# 37. OAuth Redirect URI APIs

```http
GET    /api/v1/oauth/clients/{client_id}/redirect-uris
POST   /api/v1/oauth/clients/{client_id}/redirect-uris
DELETE /api/v1/oauth/clients/{client_id}/redirect-uris/{redirect_uri_id}
```

Redirect URI validation must be exact.

---

# 38. OAuth Grant APIs

```http
GET    /api/v1/oauth/clients/{client_id}/grants
POST   /api/v1/oauth/clients/{client_id}/grants
DELETE /api/v1/oauth/clients/{client_id}/grants/{grant_id}
```

---

# 39. OAuth Scope APIs

```http
GET    /api/v1/oauth/scopes
POST   /api/v1/oauth/scopes
GET    /api/v1/oauth/scopes/{scope_id}
PATCH  /api/v1/oauth/scopes/{scope_id}
DELETE /api/v1/oauth/scopes/{scope_id}
```

OAuth scopes are not the same as ERP permissions.

Example:

```text
OAuth scope:
finance

ERP permission:
finance.invoice.approve
```

---

# 40. Authorization Code Handling

Do not expose public CRUD APIs for authorization codes.

Correct flow:

```text
/oauth/authorize
       ↓
authorization code
       ↓
/oauth/token
       ↓
access token
```

Authorization codes should be short-lived and single-use.

---

# 41. Refresh Token Handling

Do not expose raw refresh-token CRUD APIs.

Use:

```http
POST /oauth/token
POST /oauth/revoke
```

Internally support:

```text
family_id
parent_token_id
replaced_by_token_id
used_at
revoked_at
reuse detection
```

If reuse is detected, revoke the whole token family.

---

# 42. Service Account APIs

```http
GET    /api/v1/service-accounts
POST   /api/v1/service-accounts
GET    /api/v1/service-accounts/{service_account_id}
PATCH  /api/v1/service-accounts/{service_account_id}
DELETE /api/v1/service-accounts/{service_account_id}

POST /api/v1/service-accounts/{id}/activate
POST /api/v1/service-accounts/{id}/deactivate

GET  /api/v1/service-accounts/{id}/credentials
POST /api/v1/service-accounts/{id}/credentials
POST /api/v1/service-accounts/{id}/credentials/rotate
POST /api/v1/service-accounts/{id}/credentials/revoke
```

Service accounts should use client credentials, not human passwords.

---

# 43. SSO Provider APIs

```http
GET    /api/v1/sso/providers
POST   /api/v1/sso/providers
GET    /api/v1/sso/providers/{provider_id}
PATCH  /api/v1/sso/providers/{provider_id}
DELETE /api/v1/sso/providers/{provider_id}

POST /api/v1/sso/providers/{provider_id}/test
```

Supported provider families:

```text
OIDC
SAML
```

---

# 44. SSO Login APIs

OIDC:

```http
GET /api/v1/sso/{provider_id}/authorize
GET /api/v1/sso/{provider_id}/callback
```

SAML:

```http
GET  /api/v1/sso/{provider_id}/login
POST /api/v1/sso/{provider_id}/acs
GET  /api/v1/sso/{provider_id}/metadata
```

**Shipped honesty (TASK-SOR-028):** SAML ACS returns **501** (IAM-020, not implemented). `POST /api/v1/sso/providers/{id}/test` is **PROVIDER_PENDING** under pytest and never hits a live JWKS. OIDC callback code-exchange / JWKS verify is a port — pytest must not call a live IdP.

SSO identity management:

```http
GET    /api/v1/users/{user_id}/sso-identities
DELETE /api/v1/users/{user_id}/sso-identities/{identity_id}
```

Identity matching should use:

```text
provider + stable subject
```

rather than email alone.

---

# 45. Security Policy APIs

```http
GET   /api/v1/security/policies
POST  /api/v1/security/policies
GET   /api/v1/security/policies/{policy_id}
PATCH /api/v1/security/policies/{policy_id}
```

Policies can control:
- MFA
- Password
- Sessions
- IP restrictions
- Geo restrictions
- Device trust
- Login rules
- Time windows

---

# 46. Password Policy APIs

```http
GET   /api/v1/security/password-policy
POST  /api/v1/security/password-policy
PATCH /api/v1/security/password-policy
```

Example:

```json
{
  "minimum_length": 12,
  "require_uppercase": true,
  "require_lowercase": true,
  "require_number": true,
  "require_special": true,
  "password_history_count": 5
}
```

---

# 47. Time-Based Policy APIs

```http
GET    /api/v1/security/time-policies
POST   /api/v1/security/time-policies
GET    /api/v1/security/time-policies/{id}
PATCH  /api/v1/security/time-policies/{id}
DELETE /api/v1/security/time-policies/{id}
```

Use cases:
- Allowed login hours
- Blocked login hours
- Weekend restrictions
- Temporary access
- Branch-specific access windows

---

# 48. Tenant IAM Settings

Tenant master remains in ORG.

IAM configuration:

```http
GET   /api/v1/tenants/{tenant_id}/iam-settings
PATCH /api/v1/tenants/{tenant_id}/iam-settings
```

Possible settings:
- MFA required
- SSO enabled
- Authentication methods
- Session limits
- Password policy
- Login restrictions

---

# 49. Audit APIs

Read-only administrative audit APIs:

```http
GET /api/v1/audit/events
GET /api/v1/audit/events/{event_id}
GET /api/v1/users/{user_id}/audit-events
GET /api/v1/tenants/{tenant_id}/audit-events
```

Filters:

```text
actor_id
user_id
tenant_id
company_id
branch_id
event_type
action
status
ip_address
date_from
date_to
```

Audit records should be append-only.

---

# 50. Security Event APIs

```http
GET /api/v1/security/events
GET /api/v1/security/events/{event_id}
```

Examples:

```text
LOGIN_FAILED
LOGIN_SUCCESS
MFA_FAILED
TOKEN_REUSE
SUSPICIOUS_LOGIN
IMPOSSIBLE_TRAVEL
IP_BLOCKED
ACCOUNT_LOCKED
PASSWORD_ATTACK
SESSION_REVOKED
```

Security-event write operations should remain internal.

---

# 51. Admin APIs

Platform/administrative APIs:

```http
GET /api/v1/admin/users
GET /api/v1/admin/sessions
GET /api/v1/admin/security-events
GET /api/v1/admin/audit-events
```

Administrative actions:

```http
POST /api/v1/admin/users/{id}/lock
POST /api/v1/admin/users/{id}/unlock
POST /api/v1/admin/users/{id}/disable
POST /api/v1/admin/users/{id}/force-password-reset
POST /api/v1/admin/users/{id}/force-mfa
POST /api/v1/admin/users/{id}/revoke-sessions
```

Platform-admin privileges must be separate from ordinary tenant roles.

---

# 52. Internal Organization Integration

ORG owns:

```text
Tenant
Company
Branch
Department
```

IAM should use an `OrganizationGateway`.

For a distributed deployment, internal APIs can be:

```http
GET  /internal/v1/organization/tenants/{tenant_id}
GET  /internal/v1/organization/companies/{company_id}
GET  /internal/v1/organization/branches/{branch_id}
GET  /internal/v1/organization/departments/{department_id}
POST /internal/v1/organization/validate-context
```

In a modular monolith, prefer an in-process adapter instead of unnecessary HTTP calls.

---

# 53. Token Verification

Other ERP services should verify JWTs locally.

Required discovery endpoint:

```http
GET /.well-known/jwks.json
```

Services validate:

```text
Signature
Issuer
Audience
Expiration
Not-before
Key ID
```

Do not call IAM for every business API request.

Use:

```http
POST /oauth/introspect
```

when centralized introspection is specifically required.

---

# 54. Error Contract

Use one consistent error format:

```json
{
  "error": {
    "code": "AUTHENTICATION_FAILED",
    "message": "Authentication failed.",
    "details": null,
    "request_id": "..."
  }
}
```

Never expose:
- Password hashes
- OTP values
- Secrets
- Stack traces
- SQL errors
- Internal service credentials

Authentication errors should prevent account enumeration.

---

# 55. Pagination

Collection APIs should support consistent pagination.

Example:

```http
GET /api/v1/users?limit=50&cursor=...
```

Response:

```json
{
  "items": [],
  "next_cursor": "...",
  "has_more": true
}
```

Recommended:
- Default: 50
- Maximum: 200

---

# 56. Filtering

Example:

```http
GET /api/v1/users?status=ACTIVE&search=chetan&tenant_id=...&limit=50
```

Only allow explicitly supported filter parameters.

Never accept arbitrary SQL-like filters.

---

# 57. Idempotency

Use idempotency where duplicate requests can create unwanted effects.

Examples:

```text
POST /api/v1/users
POST /api/v1/users/{user_id}/roles
POST /api/v1/users/{user_id}/permissions
POST /api/v1/context/switch
POST /api/v1/auth/whatsapp/request
```

For external side effects, idempotency is especially important.

---

# 58. Request Tracing

Support:

```http
X-Request-ID: <uuid>
```

This is for observability only and must never be treated as an authenticated identity.

---

# 59. Rate Limiting

Important endpoints:

```text
POST /api/v1/auth/login
POST /api/v1/auth/password/forgot
POST /api/v1/auth/password/reset
POST /api/v1/auth/whatsapp/request
POST /api/v1/auth/whatsapp/verify
POST /api/v1/otp/request
POST /api/v1/otp/verify
POST /api/v1/mfa/challenge
POST /oauth/token
POST /oauth/authorize
```

Possible dimensions:

```text
IP
User
Identifier
Phone
Client
Tenant
```

Redis can implement rate limiting, but PostgreSQL remains the source of truth.

---

# 60. APIs That Should NOT Be Public CRUD

Do not automatically create CRUD endpoints for every IAM table.

Keep these internal:

```text
password_history
login_attempt
otp
mfa_challenge
refresh_token
authorization_code
outbox
event_delivery_state
security_event writes
```

Do not expose:

```http
GET /api/v1/refresh-tokens
POST /api/v1/refresh-tokens
GET /api/v1/password-history
POST /api/v1/otp
```

Database tables are not automatically API resources.

---

# 61. Recommended FastAPI Router Structure

```text
infrastructure/http/routers/
│
├── health.py
├── auth.py
├── registration.py
├── users.py
├── me.py
├── assignments.py
├── context.py
├── roles.py
├── permissions.py
├── resources.py
├── authorization.py
├── otp.py
├── whatsapp.py
├── mfa.py
├── webauthn.py
├── devices.py
├── sessions.py
├── oauth.py
├── oauth_clients.py
├── oauth_scopes.py
├── sso.py
├── service_accounts.py
├── security.py
├── audit.py
├── admin.py
└── internal_organization.py
```

HTTP DTOs should remain outside the domain layer.

---

# 62. Authentication Flow

```text
Frontend
   |
   | POST /api/v1/auth/login
   v
IAM
   |
   | Validate credentials
   v
Password verification
   |
   +---- MFA required ----> MFA Challenge
   |                            |
   |                            | verify
   |                            v
   |                         IAM
   |
   v
Create Session
   |
   v
Issue Access + Refresh Token
```

Access tokens should normally be short-lived, approximately 5–15 minutes depending on your security requirements.

---

# 63. Context Switch Flow

```text
Frontend
   |
   | POST /api/v1/context/switch
   v
IAM
   |
   +--> Tenant assignment
   |
   +--> Company assignment
   |
   +--> Branch assignment
   |
   +--> ORG hierarchy
   |
   +--> Roles
   |
   +--> Permissions
   |
   +--> New context
   |
   +--> New access token
   |
   v
Frontend
```

---

# 64. ERP Service Request Flow

Example:

```text
Frontend
   |
   | Bearer JWT
   v
Finance API
   |
   | Verify JWT locally
   |
   | Build RequestContext
   v
Application Service
   |
   | AuthorizationPort
   v
IAM authorization
   |
   +--> finance.invoice.approve
   |
   v
Finance Domain
```

Finance/Sales/HR/etc. must not directly import IAM SQLAlchemy models.

---

# 65. Permission Architecture

Use:

```text
Role
  ↓
Permission
  ↓
Authorization
```

And allow direct permissions only where required.

Example:

```text
User
 ├── Tenant assignment
 ├── Company assignment
 ├── Branch assignment
 ├── Role assignment
 └── Direct permission
```

Effective permission resolution must respect the active RequestContext.

---

# 66. API Security Rules

The following should be mandatory:

1. UUID identifiers.
2. JWT access tokens with asymmetric signing.
3. JWKS and signing-key rotation.
4. Short-lived access tokens.
5. Refresh-token rotation.
6. Refresh-token reuse detection.
7. Argon2id passwords.
8. Hashed OTPs.
9. Hashed reset tokens.
10. Encrypted MFA secrets.
11. Protected client secrets.
12. Exact redirect URI validation.
13. Authorization Code + PKCE.
14. Rate limiting.
15. Security audit logging.
16. Immutable security events.
17. PostgreSQL as source of truth.
18. Redis for temporary/cache/rate-limit state.
19. Domain independent of FastAPI/SQLAlchemy/Pydantic.
20. Business modules do not import IAM persistence models.
21. Server-side authorization on every protected business operation.
22. Tenant/company/branch hierarchy validation.
23. New token when context changes.
24. Separate service-account authentication.
25. No secrets in logs.
26. Anti-enumeration authentication responses.
27. Public/internal API separation.
28. Platform-admin and tenant-admin permissions are distinct.
29. Transactional outbox for IAM events.
30. Idempotent event consumers.

---

# 67. Complete Endpoint Index

## Platform

```text
GET    /health
GET    /health/live
GET    /health/ready
GET    /version
GET    /metrics
```

## Authentication

```text
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/logout-all
POST   /api/v1/auth/refresh
GET    /api/v1/auth/me
GET    /api/v1/auth/session
GET    /api/v1/auth/sessions

POST   /api/v1/auth/password/change
POST   /api/v1/auth/password/forgot
POST   /api/v1/auth/password/reset
POST   /api/v1/auth/password/verify

POST   /api/v1/auth/email/send-verification
POST   /api/v1/auth/email/verify
POST   /api/v1/auth/email/resend-verification

POST   /api/v1/auth/whatsapp/request
POST   /api/v1/auth/whatsapp/verify
POST   /api/v1/auth/whatsapp/resend
POST   /api/v1/auth/whatsapp/change-number
POST   /api/v1/auth/whatsapp/verify-number
```

## OTP

```text
POST /api/v1/otp/request
POST /api/v1/otp/verify
POST /api/v1/otp/resend
```

## Users

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/{user_id}
PATCH  /api/v1/users/{user_id}
DELETE /api/v1/users/{user_id}

POST   /api/v1/users/{user_id}/activate
POST   /api/v1/users/{user_id}/deactivate
POST   /api/v1/users/{user_id}/suspend
POST   /api/v1/users/{user_id}/restore
POST   /api/v1/users/{user_id}/lock
POST   /api/v1/users/{user_id}/unlock

GET    /api/v1/users/{user_id}/profile
PATCH  /api/v1/users/{user_id}/profile
```

## Me

```text
GET    /api/v1/me
PATCH  /api/v1/me
GET    /api/v1/me/profile
PATCH  /api/v1/me/profile
GET    /api/v1/me/tenants
GET    /api/v1/me/companies
GET    /api/v1/me/branches
GET    /api/v1/me/departments
GET    /api/v1/me/roles
GET    /api/v1/me/permissions
GET    /api/v1/me/sessions
GET    /api/v1/me/devices
GET    /api/v1/me/mfa
GET    /api/v1/me/webauthn
GET    /api/v1/me/sso-identities
```

## Assignments

```text
GET    /api/v1/users/{user_id}/tenants
POST   /api/v1/users/{user_id}/tenants
DELETE /api/v1/users/{user_id}/tenants/{tenant_id}

POST   /api/v1/users/{user_id}/tenants/{tenant_id}/activate
POST   /api/v1/users/{user_id}/tenants/{tenant_id}/deactivate

GET    /api/v1/users/{user_id}/companies
POST   /api/v1/users/{user_id}/companies
DELETE /api/v1/users/{user_id}/companies/{company_id}

GET    /api/v1/users/{user_id}/branches
POST   /api/v1/users/{user_id}/branches
DELETE /api/v1/users/{user_id}/branches/{branch_id}

GET    /api/v1/users/{user_id}/departments
POST   /api/v1/users/{user_id}/departments
DELETE /api/v1/users/{user_id}/departments/{department_id}
```

## Context

```text
GET    /api/v1/context
GET    /api/v1/context/available
POST   /api/v1/context/switch
POST   /api/v1/context/validate
```

## Roles and Permissions

```text
GET    /api/v1/roles
POST   /api/v1/roles
GET    /api/v1/roles/{role_id}
PATCH  /api/v1/roles/{role_id}
DELETE /api/v1/roles/{role_id}

POST   /api/v1/roles/{role_id}/activate
POST   /api/v1/roles/{role_id}/deactivate

GET    /api/v1/roles/{role_id}/children
POST   /api/v1/roles/{role_id}/children
DELETE /api/v1/roles/{role_id}/children/{child_role_id}

GET    /api/v1/users/{user_id}/roles
POST   /api/v1/users/{user_id}/roles
DELETE /api/v1/users/{user_id}/roles/{role_id}

GET    /api/v1/permissions
POST   /api/v1/permissions
GET    /api/v1/permissions/{permission_id}
PATCH  /api/v1/permissions/{permission_id}
DELETE /api/v1/permissions/{permission_id}

GET    /api/v1/permissions/by-code/{permission_code}

GET    /api/v1/roles/{role_id}/permissions
POST   /api/v1/roles/{role_id}/permissions
DELETE /api/v1/roles/{role_id}/permissions/{permission_id}
POST   /api/v1/roles/{role_id}/permissions/bulk

GET    /api/v1/users/{user_id}/permissions
POST   /api/v1/users/{user_id}/permissions
DELETE /api/v1/users/{user_id}/permissions/{permission_id}
POST   /api/v1/users/{user_id}/permissions/bulk

GET    /api/v1/resources
POST   /api/v1/resources
GET    /api/v1/resources/{resource_id}
PATCH  /api/v1/resources/{resource_id}
DELETE /api/v1/resources/{resource_id}

POST   /api/v1/authorization/check
POST   /api/v1/authorization/check-bulk
```

## MFA and WebAuthn

```text
GET    /api/v1/me/mfa
POST   /api/v1/me/mfa/setup
POST   /api/v1/me/mfa/verify
POST   /api/v1/me/mfa/enable
POST   /api/v1/me/mfa/disable

POST   /api/v1/me/mfa/totp/setup
POST   /api/v1/me/mfa/totp/verify
POST   /api/v1/me/mfa/totp/enable
POST   /api/v1/me/mfa/totp/disable

GET    /api/v1/me/mfa/recovery-codes
POST   /api/v1/me/mfa/recovery-codes/regenerate
POST   /api/v1/me/mfa/recovery-codes/verify

POST   /api/v1/mfa/challenge
POST   /api/v1/mfa/challenge/{challenge_id}/verify
POST   /api/v1/mfa/challenge/{challenge_id}/resend

POST   /api/v1/webauthn/register/options
POST   /api/v1/webauthn/register/verify
POST   /api/v1/webauthn/login/options
POST   /api/v1/webauthn/login/verify

GET    /api/v1/me/webauthn
DELETE /api/v1/me/webauthn/{credential_id}
```

## Devices and Sessions

```text
GET    /api/v1/me/devices
GET    /api/v1/me/devices/{device_id}
PATCH  /api/v1/me/devices/{device_id}
DELETE /api/v1/me/devices/{device_id}
POST   /api/v1/me/devices/{device_id}/trust
POST   /api/v1/me/devices/{device_id}/untrust

GET    /api/v1/me/sessions
GET    /api/v1/me/sessions/{session_id}
POST   /api/v1/me/sessions/{session_id}/revoke
DELETE /api/v1/me/sessions/{session_id}
POST   /api/v1/me/sessions/revoke-all

GET    /api/v1/users/{user_id}/sessions
POST   /api/v1/users/{user_id}/sessions/revoke-all
```

## OAuth/OIDC

```text
GET    /oauth/authorize
POST   /oauth/token
POST   /oauth/revoke
POST   /oauth/introspect
GET    /oauth/userinfo

GET    /.well-known/openid-configuration
GET    /.well-known/jwks.json

GET    /api/v1/oauth/clients
POST   /api/v1/oauth/clients
GET    /api/v1/oauth/clients/{client_id}
PATCH  /api/v1/oauth/clients/{client_id}
DELETE /api/v1/oauth/clients/{client_id}

POST   /api/v1/oauth/clients/{client_id}/activate
POST   /api/v1/oauth/clients/{client_id}/deactivate
POST   /api/v1/oauth/clients/{client_id}/secret/rotate
POST   /api/v1/oauth/clients/{client_id}/secret/revoke

GET    /api/v1/oauth/clients/{client_id}/redirect-uris
POST   /api/v1/oauth/clients/{client_id}/redirect-uris
DELETE /api/v1/oauth/clients/{client_id}/redirect-uris/{redirect_uri_id}

GET    /api/v1/oauth/clients/{client_id}/grants
POST   /api/v1/oauth/clients/{client_id}/grants
DELETE /api/v1/oauth/clients/{client_id}/grants/{grant_id}

GET    /api/v1/oauth/scopes
POST   /api/v1/oauth/scopes
GET    /api/v1/oauth/scopes/{scope_id}
PATCH  /api/v1/oauth/scopes/{scope_id}
DELETE /api/v1/oauth/scopes/{scope_id}
```

## Service Accounts

```text
GET    /api/v1/service-accounts
POST   /api/v1/service-accounts
GET    /api/v1/service-accounts/{service_account_id}
PATCH  /api/v1/service-accounts/{service_account_id}
DELETE /api/v1/service-accounts/{service_account_id}

POST   /api/v1/service-accounts/{id}/activate
POST   /api/v1/service-accounts/{id}/deactivate

GET    /api/v1/service-accounts/{id}/credentials
POST   /api/v1/service-accounts/{id}/credentials
POST   /api/v1/service-accounts/{id}/credentials/rotate
POST   /api/v1/service-accounts/{id}/credentials/revoke
```

## SSO

```text
GET    /api/v1/sso/providers
POST   /api/v1/sso/providers
GET    /api/v1/sso/providers/{provider_id}
PATCH  /api/v1/sso/providers/{provider_id}
DELETE /api/v1/sso/providers/{provider_id}

POST   /api/v1/sso/providers/{provider_id}/test

GET    /api/v1/sso/{provider_id}/authorize
GET    /api/v1/sso/{provider_id}/callback

GET    /api/v1/sso/{provider_id}/login
POST   /api/v1/sso/{provider_id}/acs
GET    /api/v1/sso/{provider_id}/metadata

GET    /api/v1/users/{user_id}/sso-identities
DELETE /api/v1/users/{user_id}/sso-identities/{identity_id}
```

## Security and Audit

```text
GET    /api/v1/security/policies
POST   /api/v1/security/policies
GET    /api/v1/security/policies/{policy_id}
PATCH  /api/v1/security/policies/{policy_id}

GET    /api/v1/security/password-policy
POST   /api/v1/security/password-policy
PATCH  /api/v1/security/password-policy

GET    /api/v1/security/time-policies
POST   /api/v1/security/time-policies
GET    /api/v1/security/time-policies/{id}
PATCH  /api/v1/security/time-policies/{id}
DELETE /api/v1/security/time-policies/{id}

GET    /api/v1/security/events
GET    /api/v1/security/events/{event_id}

GET    /api/v1/audit/events
GET    /api/v1/audit/events/{event_id}
GET    /api/v1/users/{user_id}/audit-events
GET    /api/v1/tenants/{tenant_id}/audit-events

GET    /api/v1/tenants/{tenant_id}/iam-settings
PATCH  /api/v1/tenants/{tenant_id}/iam-settings
```

## Admin

```text
GET    /api/v1/admin/users
GET    /api/v1/admin/sessions
GET    /api/v1/admin/security-events
GET    /api/v1/admin/audit-events

POST   /api/v1/admin/users/{id}/lock
POST   /api/v1/admin/users/{id}/unlock
POST   /api/v1/admin/users/{id}/disable
POST   /api/v1/admin/users/{id}/force-password-reset
POST   /api/v1/admin/users/{id}/force-mfa
POST   /api/v1/admin/users/{id}/revoke-sessions
```

## Internal Organization

```text
GET    /internal/v1/organization/tenants/{tenant_id}
GET    /internal/v1/organization/companies/{company_id}
GET    /internal/v1/organization/branches/{branch_id}
GET    /internal/v1/organization/departments/{department_id}
POST   /internal/v1/organization/validate-context
```

---

# 68. Recommended Architecture

```text
                         ┌─────────────────────┐
                         │ React / Mobile /    │
                         │ External Clients    │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    JeslotERP IAM API   │
                         │                     │
                         │ Authentication      │
                         │ Users               │
                         │ MFA                 │
                         │ WhatsApp OTP        │
                         │ Sessions            │
                         │ OAuth/OIDC          │
                         │ SSO                 │
                         │ Roles               │
                         │ Permissions         │
                         │ Authorization       │
                         └──────────┬──────────┘
                                    │
                           JWT + RequestContext
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
   ┌─────────────┐           ┌─────────────┐           ┌─────────────┐
   │ Finance     │           │ Sales       │           │ Inventory   │
   │ API         │           │ API         │           │ API         │
   └──────┬──────┘           └──────┬──────┘           └──────┬──────┘
          │                         │                         │
          └─────────────────────────┼─────────────────────────┘
                                    │
                             AuthorizationPort
                                    │
                                    ▼
                            IAM Authorization
```

The IAM platform can initially run as a JeslotERP module and later be extracted into an independent Identity Provider because business modules depend on contracts such as `AuthorizationPort` and JWT/OIDC standards rather than IAM database models.

---

# 69. Definition of Done for the API Layer

The IAM API implementation is considered production-ready when:

- Authentication works with secure password handling.
- WhatsApp OTP works through a replaceable gateway.
- MFA works.
- Passkeys/WebAuthn work.
- Sessions are tracked and revocable.
- Access tokens are short-lived.
- Refresh tokens rotate.
- Refresh-token reuse is detected.
- JWT signing keys rotate through JWKS.
- OAuth Authorization Code + PKCE works.
- OIDC discovery works.
- SSO supports OIDC as implemented; SAML ACS is 501. Live IdP/JWKS is PROVIDER_PENDING in pytest.
- Users can belong to multiple tenants.
- Company/branch assignments respect hierarchy.
- Context switching issues a new token.
- Effective roles and permissions are context-aware.
- Authorization is enforced server-side.
- Service accounts work independently of human sessions.
- Security policies are enforceable.
- Audit events are immutable.
- Security events are recorded.
- Rate limits exist on authentication/security endpoints.
- Secrets never appear in logs.
- Internal APIs are separated from public APIs.
- Business modules do not depend directly on IAM persistence models.
- Integration tests cover authentication, context switching, authorization, token rotation, MFA, WhatsApp OTP and OAuth/OIDC.
