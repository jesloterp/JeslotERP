# JeslotERP Security Platform — Developer Integration Guide

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — Alembic `f28a0b1c2d3e` / `f28b1c2d3e4f`; not Production  
**Package:** `platforms.p28_security`  
**PostgreSQL schema:** `security`  
**Depends on:** `p01_identity` (authn/authz), `p03_configuration` (secret *values* / refs)  
**Integrates with:** `p19_audit` (SIEM/audit primitives), `p22_api` (edge), optional WAF/KMS providers  
**Registry:** [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)  
**Companion:** [`SECURITY_SCHEMA.md`](SECURITY_SCHEMA.md) · [`SECURITY_API.md`](SECURITY_API.md)

### Revision history

| Version | Date | Changes |
|---|---|---|
| **1.0 Advanced** | **2026-09-12** | KMS/HSM abstraction, key lifecycle/rotation, security policy, posture, WAF contract, security events. Lean 8-table SoR (AUD-022). |

---

## 1. Purpose (enterprise)

`p28_security` is JeslotERP’s **security control plane** — named third-party products below are **orientation only** (not affiliation or compatibility; see [TRADEMARKS.md](../../TRADEMARKS.md)). Industry patterns include:

- **SAP Secure Store / SSFS + KMS** — key metadata and rotation, not login  
- **Azure Key Vault / AWS KMS / HSM** — provider-backed wrap/unwrap  
- **Salesforce Shield (partial)** — platform encryption *orchestration*, not identity  

It exists **only** for responsibilities that p01 / p03 / p19 do **not** own (audit AUD-008, registry reserved row).

### Owns

| Domain | Examples |
|---|---|
| Key metadata | Key id, purpose, algorithm, provider ref |
| Key versions | Activate / rotate / retire (no plaintext material) |
| Provider abstraction | LOCAL_DEV / AWS_KMS / AZURE_KV / HSM ports |
| Rotation jobs | Orchestrated rotate + attest |
| Security policy | Control catalog, required posture |
| Posture checks | Findings against policy |
| WAF contract | Profile + provider binding (adapter, not a WAF product) |
| Security events | Control-plane events forwarded to p19 |

### Does **not** own

| Concern | Owner |
|---|---|
| Users, sessions, MFA, JWT, RBAC | `p01_identity` |
| Secret *values* / vault blobs | `p03_configuration` |
| Immutable audit / SIEM sink | `p19_audit` |
| Record-level sharing | `p33_sharing` |
| Tenant master | `p02_organization` |

### Critical split: Identity vs Secrets vs Security vs Audit

| | **p01 Identity** | **p03 Configuration** | **p28 Security** | **p19 Audit** |
|---|---|---|---|---|
| Question | Who is the user? | What is the setting/secret value? | How are keys/policies/posture managed? | What happened, immutably? |
| Stores | Users, roles | Setting values, secret refs | Key metadata, policies | Audit events |
| GET APIs | Never return passwords | Never return plaintext secrets | Never return key material | Redacted by default |

**Rule:** p28 never returns DEK/KEK/HSM material on GET. Providers wrap/unwrap via ports. Secret *payloads* stay in p03.

---

## 2. Architectural position

```text
HTTP / internal callers
        │
        ▼
  Security application services
        │
        ├── Key lifecycle ──► KeyRepository (PostgreSQL)
        ├── Policy / posture
        └── Provider port ──► LocalDevAdapter | AwsKmsAdapter* | HsmAdapter*
                                      │
                                      ▼
                              PostgreSQL schema `security` + RLS
```

`*` External adapters are ports + deterministic test doubles until a real provider is configured. Do not claim live KMS when the environment has none.

**Hard rules**

1. PostgreSQL is the HTTP system of record.  
2. No cross-schema FKs — UUID refs to p01/p03/p19.  
3. No plaintext key material in tables or GET bodies.  
4. FORCE RLS fail-closed on tenant-scoped rows.  
5. Do not duplicate p01 auth or p19 SIEM.

---

## 3. Advanced design principles

1. **Ports over products** — KMS/HSM/WAF behind application ports.  
2. **Metadata ≠ material** — DB holds aliases, versions, status, checksums.  
3. **Rotation is a job** — activate new version, retire old after grace.  
4. **Fail-closed** — missing tenant / invalid tenant → deny.  
5. **Idempotent rotate/create** — `Idempotency-Key`.  
6. **Provider pending is honest** — status `LOCAL_DEV` / `PROVIDER_PENDING`.  
7. **Events to p19** — p28 emits; p19 stores the trail.  
8. **Lean schema** — AUD-022; no 60-table encyclopedia.  
9. **CQRS HTTP** — thin routers; services own writes.  
10. **No unsafe code execution.**

---

## 4. Core concepts

### 4.1 Key

```text
key_id, key_alias, purpose (DATA|KEK|SIGN|WAF),
algorithm, provider_key, status (ACTIVE|ROTATING|RETIRED),
current_version, tenant_id?
```

### 4.2 Key version

```text
version_id, key_id, version_no, state (PENDING|ACTIVE|RETIRED),
wrapped_ref (opaque provider handle — never raw key),
activated_at, retired_at
```

### 4.3 Provider

Allow-listed `provider_key`. Unknown provider → `SEC_PROVIDER_UNKNOWN`.

### 4.4 Rotation job

```text
PENDING → RUNNING → SUCCEEDED | FAILED
```

Concurrent rotate of the same key is rejected (`SEC_ROTATION_IN_FLIGHT`).

### 4.5 Policy / posture / WAF

Policy is a versioned control document. Posture checks evaluate against it. WAF profiles bind an external profile id — no packet inspection in-process.

---

## 5. Provider model

| Provider | Status | Notes |
|---|---|---|
| `LOCAL_DEV` | Implemented (deterministic wrap token) | Tests / local only |
| `AWS_KMS` | Live class; `PROVIDER_PENDING` without `AWS_KMS_KEY_ID` / under pytest | Never invents ciphertext |
| `AZURE_KV` | Live class; `PROVIDER_PENDING` without vault URL+name / under pytest | Never invents ciphertext |
| `HSM` | Live class; `PROVIDER_PENDING` without `HSM_PKCS11_LIB` / under pytest | Never invents ciphertext |

---

## 6. Security

### Permissions

| Code | Use |
|---|---|
| `security.key.read` | List/get key metadata |
| `security.key.manage` | Create/retire keys |
| `security.key.rotate` | Start rotation |
| `security.policy.manage` | Policies |
| `security.posture.read` | Posture findings |
| `security.waf.manage` | WAF profiles |
| `security.event.read` | Control events |
| `security.admin` | Provider admin |
| `security.*` | Wildcard |

### RLS

FORCE RLS on all tenant-scoped `sec_*` tables. System keys may have `tenant_id` NULL and require `security.admin`.

---

## 7. Module layout

```text
platforms/p28_security/
  application/
    services/security_service.py
    ports/kms.py
    permissions/catalog.py
  domain/enums.py exceptions.py
  infrastructure/
    http/… persistence/… providers/{local_dev,aws_kms,azure_kv,hsm}.py
  tests/unit/api/ engine/ module/ permissions/
```

---

## 8. Domain events

| Event | When |
|---|---|
| `security.key.created` / `retired` | Key lifecycle |
| `security.key.rotated` | Rotation succeeded |
| `security.policy.published` | Policy version |
| `security.posture.failed` | Finding |
| `security.waf.bound` | WAF profile |

Stream: `jesloterp:security:outbox`.

---

## 9. Build phases

| Phase | Deliverable |
|---|---|
| P0 | Docs (this set) |
| P1 | Skeleton, RLS, permissions |
| P2 | Keys + versions + LOCAL_DEV port |
| P3 | Rotation jobs (idempotent, concurrency) |
| P4 | Policy + posture |
| P5 | WAF contract + events |
| P6 | Tests + registry **SoR-Live** |

---

## 10. Definition of Done (enterprise)

- [x] HTTP create/list/get keys persist in PostgreSQL when session is real  
- [x] K28-33-001: GET key is DB-first on `AsyncSession` (empty catalog 404; no memory fallback)  
- [x] GET never returns key material or `_secret_plain`  
- [x] Unknown provider rejected  
- [x] Concurrent rotate of same key does not duplicate ACTIVE versions  
- [x] FORCE RLS tenant isolation tested  
- [x] Unauthenticated / wrong permission denied  
- [x] No cross-schema FKs  
- [x] p01/p03/p19 ownership not duplicated  
- [x] P28-LIVE-001: AWS KMS / Azure KV / HSM fail-closed (`PROVIDER_PENDING`); pytest never calls a vendor SDK 

---

## 11. Anti-patterns

| Don’t | Do |
|---|---|
| Store raw AES keys in Postgres | Store provider handle + metadata |
| Return secret on GET | Return alias + version + status |
| Implement login in p28 | Call p01 |
| Fake “AWS KMS live” without credentials | Port + `PROVIDER_PENDING` |
| Cross-schema FK to `identity.iam_user` | UUID `created_by` |

---

## 12. Related documents

- Schema: [`SECURITY_SCHEMA.md`](SECURITY_SCHEMA.md)  
- API: [`SECURITY_API.md`](SECURITY_API.md)  
- RTM: [`SECURITY_RTM.md`](SECURITY_RTM.md)  
- Implementation record: [`SECURITY_IMPLEMENTATION_RECORD.md`](SECURITY_IMPLEMENTATION_RECORD.md)  
- Identity: [`../01_identity/IDENTITY_GUIDE.md`](../01_identity/IDENTITY_GUIDE.md)  
- Audit: [`../19_audit/AUDIT_GUIDE.md`](../19_audit/AUDIT_GUIDE.md)  
- Registry: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md)
