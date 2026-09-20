# p26 Licensing threat review (LIC-P-I05)

**Status:** **UNSIGNED**  
**Date:** 2026-09-14  
**Scope:** `platforms.p26_licensing` commercial control plane. Not `b01`–`b17`. Not a Production sign-off.  
**Companion:** [`IDENTITY_THREAT_MODEL.md`](IDENTITY_THREAT_MODEL.md) · [`KERNEL-THREAT-REVIEW.md`](../../task/KERNEL-THREAT-REVIEW.md) · [`PRODUCTION-PROMOTION-GATE.md`](../../task/PRODUCTION-PROMOTION-GATE.md)

An agent published this review pack. A named security reviewer has **not** signed it. `THREAT_REVIEW` stays **OPEN**. Registry stays **SoR-Live**.

---

## 1. Surfaces

| Area | What was checked | Residual |
|---|---|---|
| House split | Platform catalog/assign `403` for tenant roles. Tenant JWT cannot read another tenant’s subscription (`404`, not `403`, to avoid id oracle). `licensing.*` is platform admin and **bypasses** tenant match on `get_subscription` | Live IdP role mapping not soaked |
| RLS | `apply_lic_rls` sets `app.tenant_id` / bypass GUCs. Isolation test on `license_subscription` | Suite skips if Postgres down unless `JESLOT_RLS_REQUIRED` |
| Tokens | HMAC JWT 15 min, rotate keeps previous kids. Plaintext on-prem key issue-once; SoR is `key_hash`. Machine fingerprint hashed | No KMS; `secret_ref` in-process |
| Overrides | Explicit rows; compile rank OVERRIDE > CONTRACT > ADDON > PLAN. No live row edits | Platform override permission is `licensing.admin` |
| Trust | JWT tenant authoritative; `X-Tenant-Id` never used alone | Client headers still present on some CORS policies (ignored by p26) |
| PSP | Payment refs `PROVIDER_PENDING` without keys | Live Stripe/Razorpay deferred (Phase G) |

---

## 2. STRIDE-lite

| ID | Threat | Mitigation today | Residual |
|---|---|---|---|
| T-01 | Cross-tenant subscription read | JWT + RLS + `404` | RLS skip-if-no-PG |
| T-02 | Tenant publishes catalog | House `403` | Mis-seeded `licensing.*` on a tenant user |
| T-03 | Entitlement token theft | Short TTL + rotate | Secret not in HSM |
| T-04 | Quota bypass | Same `check_feature` on usage ingest | Dunning 402/423 off until billing |
| T-05 | Fake charge ids | Fail-closed adapters | Business may still call checkout |
| T-06 | Override privilege | Admin permission + recompile | No dual-control |

---

## 3. Sign-off (blank)

| Role | Name | Date | Verdict |
|---|---|---|---|
| Security reviewer | | | OPEN |
| Platform owner | | | OPEN |

Until both rows are dated **SIGNED**, do not set registry Status to **Production**.
