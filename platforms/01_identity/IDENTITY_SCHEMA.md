# Identity Platform — Schema

**Package:** `platforms.p01_identity`  
**PostgreSQL schema:** `identity`  
**Runtime models:** `platforms/p01_identity/infrastructure/persistence/models/iam_schema.py`  
**Migrations:** `alembic/versions/86def0388fc0_*`, `c9d0e1f2a3b4_*`, `d0e1f2a3b4c5_*`

This is the Identity schema index. Column-level detail lives in the ORM models above.

---

## Conventions

| Topic | Rule |
|---|---|
| Primary keys | UUID |
| Soft delete | `is_deleted` (+ mixins where applicable) |
| Tenancy | Global user; tenant/company/branch via **assignment** tables |
| RLS | Forced on tenant-scoped tables (`c9d0e1f2a3b4`); GUCs `app.tenant_id`, `app.bypass_rls` |
| Permissions | Codes in `identity.identity_iam_permission` (seed includes `identity.*`) |

---

## Table map

### Users & profile

| Table | Purpose |
|---|---|
| `identity_iam_user` | Global user account |
| `identity_iam_user_profile` | Profile fields |
| `identity_iam_user_category` | User category catalog |
| `identity_iam_user_category_assignment` | User ↔ category |

### Org assignments

| Table | Purpose |
|---|---|
| `identity_iam_tenant_assignment` | User ↔ tenant |
| `identity_iam_company_assignment` | User ↔ company |
| `identity_iam_branch_assignment` | User ↔ branch |
| `identity_iam_department_assignment` | User ↔ department |

### Auth / credentials

| Table | Purpose |
|---|---|
| `identity_iam_password_history` | Password history |
| `identity_iam_password_reset` | Reset tokens |
| `identity_iam_email_verification` | Email verify tokens |
| `identity_iam_otp` | OTP challenges |
| `identity_iam_login_attempt` | Login attempt log |
| `identity_iam_password_policy` | Password policy (tenant-scoped) |

### MFA / WebAuthn / devices

| Table | Purpose |
|---|---|
| `identity_iam_mfa` | MFA enrollment |
| `identity_iam_mfa_recovery_code` | Recovery codes |
| `identity_iam_mfa_challenge` | MFA challenges |
| `identity_iam_webauthn_credential` | Passkeys |
| `identity_iam_device` | Trusted devices |

### OAuth / OIDC / sessions

| Table | Purpose |
|---|---|
| `identity_iam_oauth_client` | OAuth clients |
| `identity_iam_oauth_client_redirect_uri` | Redirect URIs |
| `identity_iam_oauth_client_grant` | Allowed grants |
| `identity_iam_api_scope` | API scopes |
| `identity_iam_oauth_client_scope` | Client ↔ scope |
| `identity_iam_oauth_authorization_code` | Auth codes |
| `identity_iam_refresh_token` | Refresh tokens |
| `identity_iam_session` | Sessions |
| `identity_iam_access_token` | Access token records |
| `identity_iam_api_token` | API tokens |
| `identity_iam_api_client` | API clients |
| `identity_iam_api_key` | API keys |

### RBAC

| Table | Purpose |
|---|---|
| `identity_iam_resource` | Resources |
| `identity_iam_permission` | Permission catalog |
| `identity_iam_action` | Actions |
| `identity_iam_role` | Roles |
| `identity_iam_role_hierarchy` | Role parents |
| `identity_iam_role_permission` | Role ↔ permission |
| `identity_iam_user_role` | User ↔ role |
| `identity_iam_user_permission` | Direct user permission |
| `identity_iam_data_permission` | Data-level permissions |

### Service accounts / SSO / federation

| Table | Purpose |
|---|---|
| `identity_iam_service_account` | Service accounts |
| `identity_iam_service_account_permission` | SA permissions |
| `identity_iam_sso_provider` | SSO providers |
| `identity_iam_sso_identity` | Linked SSO identities |
| `identity_iam_federation` | Federation config |
| `identity_iam_external_identity` | External identities |

### Security / audit / settings

| Table | Purpose |
|---|---|
| `identity_iam_security_policy` | Security policies |
| `identity_iam_time_based_policy` | Time-based access |
| `identity_iam_audit_event` | Audit events |
| `identity_iam_security_event` | Security events |
| `identity_iam_tenant_setting` | Tenant IAM settings |
| `identity_iam_feature_flag` | Feature flags |
| `identity_iam_delegation` | Delegations |
| `identity_iam_impersonation` | Impersonation records |
| `identity_iam_outbox_event` | Transactional outbox |

---

## Related docs

| File | Role |
|---|---|
| [`IDENTITY_GUIDE.md`](IDENTITY_GUIDE.md) | Developer guide |
| [`IDENTITY_API.md`](IDENTITY_API.md) | API endpoints |
| [`../FUTURE_REFERENCE_P01_P02_P03.md`](../../FUTURE_REFERENCE_P01_P02_P03.md) | Post-remediation reference |
