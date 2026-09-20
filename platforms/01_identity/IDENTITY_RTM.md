# Identity Platform — RTM

**Platform:** `p01_identity`  
**Last reviewed:** 2026-09-12  
**Test command:** `pytest platforms/p01_identity/tests -q --tb=short` → **83 passed**, 1 failed (`test_alembic_head_is_latest`: local PG still `f33b1c2d3e4f`, script head `f02c1d2e3f40` from TASK-SOR-027). New file `test_sso_mfa_webauthn_paths.py` → **14 passed**.

| ID | Source | Requirement | Implementation | Status | Evidence |
|---|---|---|---|---|---|
| IAM-SOR-28a | TASK-SOR-028 | MFA TOTP verify/enable threat paths | shipped `/me/mfa` handlers | Implemented | `test_sso_mfa_webauthn_paths` |
| IAM-SOR-28b | TASK-SOR-028 | SSO no silent email link; SAML not shipped | `_resolve_or_provision_user` + ACS 501 | Implemented | same |
| IAM-SOR-28c | TASK-SOR-028 | Live IdP never called in pytest | `PROVIDER_PENDING` network helpers | Implemented | same |
| IAM-SOR-28d | TASK-SOR-028 | WebAuthn options + fail-closed verify | shipped webauthn router | Implemented | same |
| IAM-HYG-020 | AUD-020 | GUIDE package path = `platforms.p01_identity` | IDENTITY_GUIDE header + §8 tree | Implemented | `test_hyg020_identity_guide_package` |

**Coverage note:** Integration tests exercise shipped handlers. Pytest never hits a live OIDC JWKS or authenticator vendor. SAML ACS stays 501. Not Production.
