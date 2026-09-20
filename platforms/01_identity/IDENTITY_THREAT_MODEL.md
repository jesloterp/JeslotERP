# Identity threat model (PROD-KERN-001)

**Status:** **UNSIGNED** — formal artifact only. Gate `THREAT_REVIEW` stays **OPEN**.  
**Date:** 2026-09-12  
**Scope:** `platforms.p01_identity` plus consumers of its tokens. Not a Production sign-off.

This document is the kernel threat model for IAM. It is **not** a signed review. Do not flip registry Production from this file.

**Public note:** no credentials, hosts, or customer data belong in this file. Residual risks are architectural, not an exploit cookbook.

---

## 1. Assets

| Asset | Why it matters |
|---|---|
| Credentials / Argon2id hashes | Account takeover |
| Session + refresh-token family | Replay / family theft |
| Access token (JWT) | Impersonation across platforms |
| MFA / WebAuthn / recovery codes | Second-factor bypass |
| User profile PII | Disclosure (FLS) |
| Tenant / company / branch assignments | Cross-tenant access |
| OAuth client secrets / SAML certs | Federation abuse |
| RLS GUCs (`app.tenant_id`, …) | Isolation failure |

---

## 2. Trust boundaries

1. Unauthenticated public auth endpoints (`/auth/login`, OTP, reset).  
2. Bearer-authenticated `/api/v1/*` — context from **verified token**, not `X-Tenant-ID`.  
3. Internal `/internal/v1/*` — service-to-service; still no invented live IdP.  
4. Postgres + FORCE RLS — tenant isolation is a DB gate, not an app filter.  
5. p05 field-security declarations — p01 **enforces** on profile GET/PATCH when declared; no declaration → skip.

---

## 3. Threats (STRIDE-lite)

| ID | Threat | Mitigation today | Residual |
|---|---|---|---|
| T-01 | Password stuffing | Rate limits + generic errors; pytest does not invent live IdP | Live rate-limit soak unsigned |
| T-02 | Refresh reuse | Family revoke on reuse | Needs soak + signed review |
| T-03 | Tenant header spoof | Token is SoR for tenant_id | Mis-issued tokens |
| T-04 | Privilege escalation | RBAC + DENY precedence; platform admin is a distinct role | Admin session length |
| T-05 | Profile PII leak | p05 FLS on `identity.user` GET/PATCH | Only when a policy is declared |
| T-06 | MFA bypass | Shipped paths tested; SAML ACS 501 under pytest | Live IdP/JWKS not claimed |
| T-07 | Secret logging | GUIDE forbids password/OTP/token logs | Ops review unsigned |
| T-08 | RLS bypass | FORCE GUC on identity session | Suite skips if Postgres down |
| T-09 | SSO assertion forge | SAML ACS 501 in pytest; live parse requires cert | Unsigned live ACS |

---

## 4. What this is not

- Not a signed `THREAT_REVIEW`.  
- Not permission to set registry **Production**.  
- Not a business-ERP threat model (`b01`–`b17`).  
- Not a claim that live OIDC/SAML/SMTP/KMS ran in this environment.

---

## 5. Sign-off (blank)

| Role | Name | Date | Verdict |
|---|---|---|---|
| Security reviewer | | | OPEN |
| Platform owner | | | OPEN |

Until both rows are dated **SIGNED**, `PRODUCTION-PROMOTION-GATE.md` stays `THREAT_REVIEW: OPEN`.
