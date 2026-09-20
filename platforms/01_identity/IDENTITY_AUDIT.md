# JeslotERP IAM PRODUCTION AUDIT

> **Historical reference (2026-09-08).** Keep for audit trail. Many CRITICAL/HIGH items were remediated afterward — see `docs/FUTURE_REFERENCE_P01_P02_P03.md` and current code. Do not treat scores in this file as live production status.

---


**Scope:** `platforms/p01_identity` (+ `shared/settings.py`, Alembic)  
**Mode:** READ-ONLY — no code modified  
**Date:** 2026-09-08  
**Method:** Static architecture + adversarial scenario review against live source

---

## Executive Summary

| Metric | Value |
|---|---|
| **Overall Score** | **54 / 100** |
| Verdict | **Looks complete; not production-safe yet** |
| CRITICAL | 6 |
| HIGH | 14 |
| MEDIUM | 12 |
| LOW / INFO | 6 |

Endpoints and schemas are largely present. Several security controls are real (RS256 JWKS, refresh reuse detection, session revoke in `require_auth`, RBAC DENY/hierarchy, lean JWT, password history, policy MFA). Gaps that matter for production are **privilege / tenant-admin confusion**, **locked users retaining live sessions**, **SSO email auto-link**, **role privilege carry across tenants**, **Alembic↔model drift**, **HS256 legacy decode weakening**, **fat routers / unused CQRS**, **outbox without consumer**, and **almost no adversarial tests**.

*Supplemental findings from deep-dive explorers merged below (IAM-027+).*

---

## Scorecard

| Area | Status | Severity |
|---|---|---|
| Architecture (DDD/CQRS/hexagonal) | FAIL | High |
| Database / secrets-at-rest | FAIL | Critical |
| API contract coverage | PASS | — |
| Authentication | WARN | High |
| JWT / Token security | WARN | High |
| Refresh rotation | PASS | — |
| Multi-tenancy | FAIL | Critical |
| Context switching | WARN | High |
| Authorization / RBAC | FAIL | Critical |
| MFA | WARN | Medium |
| WhatsApp / Email OTP | WARN | High |
| WebAuthn | WARN | Medium |
| OAuth / OIDC | WARN | Medium |
| SSO | FAIL | Critical |
| Service accounts | WARN | Medium |
| Sessions | WARN | Critical |
| Rate limiting | WARN | Medium |
| Audit logging | PASS | — |
| Outbox / Events | WARN | High |
| Testing | FAIL | High |
| Deployment / Secrets / Ops | FAIL | High |

\*Passwords/tokens hashed and TOTP encrypted in current ORM write paths; Alembic drift + OTP SHA-256 + default secrets still FAIL overall.

---

## CRITICAL FINDINGS — REMEDIATION STATUS (2026-09-08)

| ID | Status | Fix summary |
|---|---|---|
| IAM-001 | **FIXED** | `require_auth` loads `IAMUser` and rejects inactive/locked/deleted; lock/suspend/deactivate/disable call `revoke_all_user_access` |
| IAM-002 | **FIXED** | `tenant_admin` removed from platform admin set; `/api/v1/admin/*` requires `require_platform_admin`; tenant admins scoped via `assert_can_manage_user` |
| IAM-003 | **FIXED** | `resolve_scope` rejects foreign `tenant_id` for non-platform admins; tenant assign forces actor tenant |
| IAM-004 | **FIXED** | SSO no longer silent-links by email; existing email → 403; JIT creates new users only |
| IAM-027 | **FIXED** | `assert_can_assign_role` blocks system/platform roles for non-platform admins |
| IAM-028 | **FIXED (migration)** | Alembic `b2c3d4e5f6a7` renames drifted secret columns idempotently |

Also: roles list IDOR gated; `resolve_authz` scopes roles by tenant/company/branch when context provided.

---

## CRITICAL FINDINGS (original audit)

### IAM-001 — Locked / suspended / deleted users keep valid access tokens
- **Severity:** CRITICAL  
- **Category:** SESSION_SECURITY / AUTHENTICATION  
- **File:** `platforms/p01_identity/infrastructure/http/dependencies/auth.py` (`require_auth`, ~49–104)  
- **Also:** `users_lifecycle.lock_user` / `admin_lock_user` (~225–247, ~153–167) — set `status=LOCKED` but do **not** revoke sessions/refresh tokens  
- **Wrong:** `require_auth` validates JWT + session row only; never loads `IAMUser` to check `is_deleted`, `status`, `is_locked`, `is_active`.  
- **Danger:** Account lock/suspend/delete is ineffective until access token + session natural expiry.  
- **Exploit:** User is locked by admin → continues calling APIs with existing Bearer token while session remains ACTIVE.  
- **Expected:** Lock/suspend/delete must revoke all sessions + refresh tokens; `require_auth` must reject inactive/deleted users.  
- **Fix:** On lock/suspend/delete, revoke sessions/refresh; in `require_auth`, load user and reject non-ACTIVE.  
- **Test missing:** Yes — locked-user token reuse.

### IAM-002 — `tenant_admin` treated as platform admin (cross-tenant privilege)
- **Severity:** CRITICAL  
- **Category:** AUTHORIZATION / MULTI_TENANCY  
- **File:** `platforms/p01_identity/infrastructure/http/routers/_common.py` (`require_admin`, ~23–28)  
- **Wrong:** `ADMIN_ROLES` includes `tenant_admin`. Any JWT with that role string passes admin gates used for global user lifecycle, permission grants, tenant assignment, etc.  
- **Danger:** Tenant admin becomes de-facto platform admin.  
- **Exploit:** Tenant A admin calls `POST /users/{any}/permissions` or `POST /users/{any}/tenants` with another `tenant_id` in body (`resolve_scope` trusts DTO).  
- **Expected:** Platform vs tenant admin separation; tenant admin scoped to own tenant only.  
- **Fix:** Split `require_platform_admin` / `require_tenant_admin(tenant_id)`; never put `tenant_admin` in global admin set.  
- **Test missing:** Yes — tenant admin cannot act on other tenants.

### IAM-003 — Admin assignment APIs trust `tenant_id` from request body without membership check
- **Severity:** CRITICAL  
- **Category:** MULTI_TENANCY  
- **File:** `_common.resolve_scope` (~31–37); used by `roles.assign_user_role`, `users_lifecycle.grant_user_permission`, tenant/company assign, etc.  
- **Wrong:** `dto.tenant_id or current_user.tenant_id` — no check that caller is assigned to / authorized for that tenant. Combined with IAM-002 this is cross-tenant escalation.  
- **Exploit:** Admin JWT + crafted `tenant_id` in body → grant roles/permissions/assignments into foreign tenant context.  
- **Expected:** Scope must be caller’s allowed tenants (or platform-admin-only for cross-tenant).  
- **Fix:** Validate assignment/ORG ownership before write.  
- **Test missing:** Yes.

### IAM-004 — SSO auto-links by email alone (account takeover)
- **Severity:** CRITICAL  
- **Category:** SSO  
- **File:** `platforms/p01_identity/infrastructure/http/routers/sso.py` (`_resolve_or_provision_user`, ~522–565)  
- **Wrong:** If no `(provider, subject)` mapping, first existing user with matching `email_normalized` is linked and given tenant assignment.  
- **Danger:** Compromised / malicious IdP issuing victim email claim takes over existing account.  
- **Exploit:** Attacker IdP returns `email=victim@corp.com` → SSO callback links attacker subject to victim user → login as victim.  
- **Expected:** Link only after verified binding (logged-in link flow) or require email_verified + explicit policy; never silent link from untrusted IdP.  
- **Fix:** Disable auto-link by default; require authenticated link or admin approval.  
- **Test missing:** Yes.

### IAM-027 — Tenant admin can self-assign `platform_admin` / system roles
- **Severity:** CRITICAL  
- **Category:** AUTHORIZATION / RBAC  
- **File:** `roles.py` `assign_user_role` (~234–247); `_get_role` does not check `is_system` / privilege ceiling  
- **Wrong:** Any `require_admin` caller can assign any role including platform/system roles to self or others.  
- **Danger:** Combines with IAM-002 → immediate platform takeover.  
- **Exploit:** Tenant admin `POST /users/{self}/roles` with `platform_admin` role_id.  
- **Expected:** Privilege ceiling; system roles only by platform admin; forbid assigning roles above actor’s tier.  
- **Fix:** Gate on `is_system` / role code allow-list; block self-elevation.  
- **Test missing:** Yes.

### IAM-028 — Alembic initial migration badly drifted from `iam_schema` models
- **Severity:** CRITICAL  
- **Category:** DATABASE  
- **Files:** `iam_schema.py` vs `alembic/versions/2f7eb1ff86a8_*.py` (+ ad-hoc `apply_iam_schema.py`)  
- **Wrong:** Column renames not migrated (e.g. MFA `secret_key` vs `secret_encrypted`; reset `reset_token_hash` vs `token_hash`; SSO `client_secret_hash` vs `client_secret_encrypted`; user still has tenant columns in migration). Refresh family fields may be missing from initial migration.  
- **Danger:** ORM writes to columns that may not exist; old plaintext/hash columns may remain in DB dumps; prod deploy divergence.  
- **Expected:** Alembic revisions match models as single source of truth.  
- **Fix:** Proper rename/alter migrations; CI schema-diff.  
- **Test missing:** Yes.

---

## HIGH FINDINGS — REMEDIATION STATUS (2026-09-08)

| ID | Status | Fix summary |
|---|---|---|
| IAM-005 | **FIXED** | HS256 never skips aud/iss; disabled in production by default (`ALLOW_LEGACY_HS256`) |
| IAM-006 | **FIXED** | Default `ACCESS_TOKEN_EXPIRE_MINUTES=15` (env may override) |
| IAM-008 | **FIXED (minimal)** | `drain_outbox` + `POST /api/v1/admin/outbox/drain` (log publisher stub) |
| IAM-010 | **FIXED (partial)** | `validate_runtime_secrets()` fails fast in production for weak JWT / missing ENCRYPTION_KEY / default admin password |
| IAM-012 | **FIXED** | Shared `mfa_gate_for_login` on password, email OTP, WhatsApp login |
| IAM-032 | **FIXED** | OTP/token hashes use HMAC-SHA256 with pepper; legacy SHA-256 still accepted for verify |
| IAM-033 | **FIXED** | ENCRYPTION_KEY required in production; JWT derivation only outside prod |
| IAM-011 | **FIXED (partial)** | Adversarial unit suite in `test_adversarial_security.py` covers CRITICAL/HIGH control regressions |
| IAM-007 | **FIXED** | `_validate_scope` / `SwitchContextHandler` calls OrgGateway on validate+switch |
| IAM-009 | **FIXED (partial)** | Context router + password login wired to application CQRS handlers; other fat routers still pending |
| IAM-017 | **FIXED** | Context switch revokes previous ACTIVE session (`CONTEXT_SWITCH`) |
| IAM-013 | **FIXED** | OAuth PKCE S256-only by default (`ALLOW_PKCE_PLAIN` blocked in prod) |
| IAM-014 | **FIXED** | Prod requires `RATE_LIMIT_ENABLED`; fail-closed without Redis in prod |
| IAM-015 | **FIXED** | `require_admin` / `require_platform_admin` use live DB roles via `resolve_authz` |
| IAM-018 | **FIXED** | Service-account / SSO routers use shared `_common.require_admin` |
| IAM-019 | **FIXED (ops hook)** | `KeyManager.rotate/reload` + `POST /api/v1/admin/jwt/rotate|reload` |
| IAM-020 | **DOCUMENTED** | SAML ACS remains 501 — OIDC only; not production for SAML |

Still open: IAM-009 remaining fat routers (users/SSO/OAuth/security); IAM-011 E2E; IAM-021 ops (Celery/TLS/backups).


---

## HIGH FINDINGS

### IAM-005 — HS256 legacy fallback can skip aud/iss verification
- **Severity:** HIGH  
- **Category:** TOKEN_SECURITY  
- **File:** `infrastructure/security/crypto.py` `_decode_hs256_legacy` (~43–60), `decode_jwt` (~63–84)  
- **Wrong:** On `MissingRequiredClaimError`, re-decodes with `verify_aud=False, verify_iss=False`. Default `JWT_SECRET_KEY` is a known string in `shared/settings.py`.  
- **Danger:** If HS256 secret is known/leaked, forge tokens without iss/aud.  
- **Expected:** Reject legacy tokens missing iss/aud; disable HS256 in production.  
- **Fix:** Env flag `ALLOW_LEGACY_HS256=false` in prod; never disable aud/iss.  
- **Test missing:** Yes.

### IAM-006 — Access token lifetime 60 minutes (guide: 5–15)
- **Severity:** HIGH  
- **Category:** TOKEN_SECURITY  
- **File:** `shared/settings.py` `ACCESS_TOKEN_EXPIRE_MINUTES = 60`  
- **Wrong:** Long-lived access tokens increase stolen-token window (esp. with IAM-001).  
- **Expected:** Short-lived access + refresh rotation.  
- **Fix:** Default 10–15 min in prod settings.  
- **Test missing:** Config assertion.

### IAM-007 — Context switch skips ORG hierarchy gateway
- **Severity:** HIGH  
- **Category:** MULTI_TENANCY  
- **File:** `routers/context.py` `_validate_scope` / `switch_context` (~64–118)  
- **Wrong:** Validates IAM assignment rows only; does not call `OrganizationGateway` to verify company∈tenant / branch∈company in ORG.  
- **Danger:** Stale/orphan assignment rows can issue context tokens for invalid org trees.  
- **Expected:** Assignment + ORG hierarchy checks.  
- **Fix:** Call org gateway in `_validate_scope`.  
- **Test missing:** Yes.

### IAM-008 — Outbox writer without consumer/dispatcher
- **Severity:** HIGH  
- **Category:** OUTBOX / EVENTS  
- **File:** `messaging/outbox/writer.py`; model `IAMOutboxEvent`  
- **Wrong:** Events enqueued PENDING; no poller/publisher/idempotent consumer.  
- **Danger:** Downstream systems never see login/lock/role events; false sense of event-driven security.  
- **Expected:** Dispatcher + idempotent consumers.  
- **Fix:** Add drain worker; until then document as incomplete.  
- **Test missing:** Yes.

### IAM-009 — Application CQRS layer unused (architecture theater)
- **Severity:** HIGH  
- **Category:** CQRS / HEXAGONAL_ARCHITECTURE / DDD  
- **Evidence:** Zero router imports from `platforms.p01_identity.application`  
- **Wrong:** Fat routers: `users_lifecycle.py` ~1000 LOC, `sso.py` ~587, `oauth.py` ~561, `security.py` ~560, `auth.py` ~464. Domain events exist but are not emitted from login paths.  
- **Danger:** Business rules duplicated; hard to test; authorization drifts.  
- **Expected:** Thin routers → application commands/queries → ports.  
- **Fix:** Migrate login/authz/context first.  
- **Test missing:** Architecture tests.

### IAM-010 — Hardcoded / default production secrets in settings
- **Severity:** HIGH  
- **Category:** SECRETS  
- **File:** `shared/settings.py` — `JWT_SECRET_KEY`, `SEED_ADMIN_PASSWORD=admin123`, `COUNTRYSTATECITY_API_KEY`, DB password default  
- **Wrong:** Insecure defaults ship in repo.  
- **Danger:** Deploy without override → trivial compromise.  
- **Expected:** Fail-fast if secrets not set when `APP_ENV=production`.  
- **Fix:** Required env validation for prod.  
- **Test missing:** Startup guard test.

### IAM-011 — Almost no security/integration tests
- **Severity:** HIGH  
- **Category:** TESTING  
- **Evidence:** Only 4 unit test files; no E2E for tenant escape, IDOR, refresh reuse, MFA bypass, OAuth.  
- **Danger:** Regressions on CRITICAL paths go unnoticed.  
- **Expected:** Adversarial integration suite for scenarios 1–32.  
- **Fix:** Add pytest integration with DB fixtures.  
- **Test missing:** Entire adversarial suite.

### IAM-012 — OTP / WhatsApp login may bypass password MFA policy path
- **Severity:** HIGH  
- **Category:** MFA / WHATSAPP_OTP  
- **File:** `otp.py` LOGIN purpose (~65–73); `whatsapp.py` LOGIN (~83–92)  
- **Wrong:** Issues session tokens after OTP without applying `IAMSecurityPolicy.require_mfa` the way password login does.  
- **Danger:** MFA policy enforced on password login, bypassed via OTP login channel.  
- **Expected:** Same MFA policy for all primary authentication methods (or document OTP as second factor only).  
- **Fix:** Share MFA policy gate in token issuance path.  
- **Test missing:** Yes.

### IAM-029 — `resolve_authz` does not filter roles by tenant (privilege carry)
- **Severity:** HIGH  
- **Category:** MULTI_TENANCY / RBAC  
- **File:** `_auth_helpers.py` `resolve_authz` (~143–152) — role query has no tenant/company/branch filter; direct perms partially filtered (~222–227)  
- **Wrong:** Admin role granted in Tenant A remains in JWT after context switch to Tenant B.  
- **Danger:** Cross-tenant privilege carry; amplifies IAM-002.  
- **Exploit:** Hold `tenant_admin` in T1 → switch to T2 → still admin in JWT roles.  
- **Expected:** Filter `IAMUserRole` by current context before JWT/admin checks.  
- **Fix:** Scope role query; re-resolve admin from DB on mutations.  
- **Test missing:** Yes.

### IAM-030 — `GET /users/{user_id}/roles` IDOR
- **Severity:** HIGH  
- **Category:** AUTHORIZATION / API_SECURITY  
- **File:** `roles.py` `user_roles` (~223–231)  
- **Wrong:** Any authenticated user can list another user’s roles; no self-or-admin check.  
- **Danger:** RBAC disclosure aids targeting.  
- **Expected:** Same pattern as profile (`user_id == current` or admin).  
- **Fix:** Add gate.  
- **Test missing:** Yes.

### IAM-031 — Internal org API fail-open when token unset
- **Severity:** HIGH  
- **Category:** API_SECURITY  
- **File:** `internal_org.py` `_require_internal` (~19–22)  
- **Wrong:** If `INTERNAL_API_TOKEN` / `INTERNAL_SERVICE_TOKEN` unset, request is allowed without auth.  
- **Danger:** Unauthenticated org validate/metadata if route exposed.  
- **Expected:** Fail closed when expected token missing (except explicit local-only mode).  
- **Fix:** Reject 401 when expected is empty in non-dev.  
- **Test missing:** Yes.

### IAM-032 — OTP hashed with unsalted SHA-256 (offline brute-force)
- **Severity:** HIGH  
- **Category:** PASSWORD_SECURITY / WHATSAPP_OTP  
- **File:** `token_utils.hash_token` + `_otp_helpers`  
- **Wrong:** 6-digit OTP → SHA-256; 1e6 space trivial after DB leak.  
- **Danger:** Dump `identity_iam_otp` → recover codes within TTL.  
- **Expected:** HMAC with server pepper (or equivalent).  
- **Fix:** Peppered HMAC; keep rate limits.  
- **Test missing:** Yes.

### IAM-033 — Fernet encryption key derived from JWT secret when blank
- **Severity:** HIGH  
- **Category:** SECRETS  
- **File:** `fernet_encryption.py`; `settings.ENCRYPTION_KEY=""`  
- **Wrong:** One secret compromise decrypts TOTP + SSO client secrets.  
- **Expected:** Independent required `ENCRYPTION_KEY` in production.  
- **Fix:** Fail-fast if unset in prod.  
- **Test missing:** Yes.

---

## MEDIUM FINDINGS

### IAM-013 — OAuth allows `code_challenge_method=plain`
- **Category:** OAUTH  
- **File:** `oauth.py` ~220–224  
- **Risk:** Weaker PKCE; prefer S256-only for public clients.

### IAM-014 — Rate limiter fails open when `RATE_LIMIT_ENABLED=False`; memory fallback not cross-instance
- **Category:** RATE_LIMITING  
- **File:** `ratelimit/limiter.py` ~49–70  
- **Risk:** Multi-worker brute force bypass without Redis.

### IAM-015 — `require_admin` based on JWT role claims, not live DB roles
- **Category:** AUTHORIZATION  
- **File:** `_common.require_admin`  
- **Risk:** Stale elevation until token expiry (roles in JWT). Prefer DB check for admin mutations.

### IAM-016 — Domain layer clean of FastAPI/SQLAlchemy (PASS architecture purity) but unused
- **Category:** DDD  
- **Info:** Domain imports are stdlib-only — good structure, weak wiring.

### IAM-017 — Context switch creates new session; old session remains ACTIVE
- **Category:** SESSION_SECURITY  
- **File:** `context.switch_context`  
- **Risk:** Session sprawl / parallel contexts; may be intentional — document or revoke prior.

### IAM-018 — Service-account admin role set narrower than `_common.ADMIN_ROLES` (inconsistency)
- **Category:** SERVICE_ACCOUNTS  
- **File:** `service_accounts.py` `_ADMIN_ROLES` vs `_common.ADMIN_ROLES`  
- **Risk:** Confusion; still uses JWT roles.

### IAM-019 — No automated JWT key-rotation job
- **Category:** TOKEN_SECURITY / DEPLOYMENT  
- **JWKS** exists; rotation ops missing.

### IAM-020 — SAML ACS still unimplemented (501)
- **Category:** SSO  
- **Deferred by product** — document as not production if SAML required.

### IAM-021 — Celery / Redis workers / monitoring / TLS / backups not evidenced in Identity module
- **Category:** DEPLOYMENT / OBSERVABILITY  

---

## LOW / INFO

### IAM-022 — No `X-Tenant-ID` header trust found (PASS)
### IAM-023 — Refresh reuse detection implemented (PASS) — `auth.refresh` ~250–255  
### IAM-024 — Password hashing via pwdlib recommended (Argon2 family) (PASS)  
### IAM-025 — Lean JWT roles-only + `resolve_authz` for checks (PASS after gap fixes)  
### IAM-026 — Session revoke enforced in `require_auth` when `session_id` present (PASS for logout)

---

## Attack scenario matrix (static)

| # | Scenario | Result | Evidence |
|---|---|---|---|
| 1–3 | Cross tenant/company/branch via headers | **PASS** | No X-Tenant headers |
| 4–6 | Tamper JWT claims without key | **PASS** | RS256 signature |
| 7 | URL user_id IDOR (profile/perms) | **PASS*** | Self or `require_admin` (*broken by IAM-002) |
| 7b | `GET /users/{id}/roles` IDOR | **FAIL** | IAM-030 |
| 8–9 | Access/revoke other session | **PASS** | `sessions.py` ownership checks |
| 10–11 | Self-assign role/permission | **FAIL** | IAM-027 (admin path) |
| 12 | Tenant admin → platform ops | **FAIL** | IAM-002 / IAM-027 |
| 13–14 | Suspended/deleted user token | **FAIL** | IAM-001 |
| 15 | Revoked session token | **PASS** | `require_auth` |
| 16 | Expired access token | **PASS** | decode rejects expired |
| 17–18 | Wrong aud/iss (RS256) | **PASS**; HS256 legacy **FAIL path** | IAM-005 |
| 19 | Refresh reuse | **PASS** | family revoke |
| 20–22 | OAuth code reuse / redirect / PKCE | **PASS** (plain PKCE WARN) | oauth.py |
| 23 | Client secret leakage | **WARN** | hashed at rest; defaults elsewhere |
| 24–27 | OTP brute / replay / enum | **WARN** | rate limits OK; at-rest hash weak (IAM-032) |
| 28 | MFA bypass via OTP login | **FAIL** | IAM-012 |
| 29 | Recovery reuse | **PASS** | `used_at` set |
| 30 | SSO account takeover | **FAIL** | IAM-004 |
| 31 | WebAuthn confusion | **WARN** | not deep-fuzzed this pass |
| 32 | Service account escalation | **WARN** | admin-gated; JWT role trust |
| — | Role privilege carry after context switch | **FAIL** | IAM-029 |
| — | Internal org unauthenticated | **FAIL** | IAM-031 (if token unset) |

---

## What looks real (not theater)

- RS256 signing + JWKS path  
- Refresh rotation + reuse → revoke family  
- Session status check on protected requests  
- RBAC hierarchy + DENY + expiry in `resolve_authz`  
- Password history enforcement  
- Policy MFA on **password** login  
- OrgGateway fail-closed setting  
- OAuth redirect allow-list + code single-use + PKCE  
- OTP request anti-enumeration message  
- Audit / security event writes on sensitive paths  

## What looks complete but isn’t safe enough

- “Admin” guard without tenant scope  
- Lock/suspend without session kill + user re-check  
- SSO email auto-link  
- CQRS folders without runtime use  
- Outbox rows without dispatcher  
- Endpoint count ≠ adversarial coverage  

---

## Recommended fix order (do not implement in this audit)

1. IAM-001 + IAM-002 + IAM-003 + IAM-027 — lock kills sessions; split platform vs tenant admin; scope writes; block system-role assign  
2. IAM-004 — SSO linking policy  
3. IAM-028 — Alembic ↔ model alignment (deploy safety)  
4. IAM-029 + IAM-030 + IAM-031 — tenant-scoped roles; roles IDOR; internal fail-closed  
5. IAM-005 + IAM-006 + IAM-010 + IAM-032 + IAM-033 — token/secret/OTP hardening  
6. IAM-012 — unify MFA policy on all login methods  
7. IAM-007, IAM-008, IAM-011 — ORG check, outbox drain, adversarial tests  
8. IAM-009 CQRS migration (after security)

---

## Deep-dive supplements

Additional evidence merged from explorers:
- [Explore CQRS and outbox](83e201a3-9ae4-4686-a3b1-07449211c4df) — outbox write-only; CQRS unwired; no Celery app  
- [IAM authz tenant audit](985f25a3-534a-4bcf-8430-1d8c3a8c7a67) — tenant_admin, roles IDOR, resolve_authz tenant filter, internal fail-open  
- [IAM architecture DB audit](32cda384-a933-4fc0-9cd0-981b3eaf67ea) — Alembic drift, OTP SHA-256, soft-delete uniques, encryption key coupling  

---

## Audit 2 note

This report is **Audit 1 (static)**. Full **Audit 2 (live adversarial)** requires running API + DB and executing forged-token / cross-tenant HTTP cases. Do that before any production cutover.
