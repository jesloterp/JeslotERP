# JeslotERP Security Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — lean SoR (AUD-022); Alembic `f28a0b1c2d3e` / `f28b1c2d3e4f`  
**Package:** `platforms.p28_security`  
**PostgreSQL schema:** `security`  
**Companion:** [`SECURITY_GUIDE.md`](SECURITY_GUIDE.md) · [`SECURITY_API.md`](SECURITY_API.md)

> Runtime models: `platforms/p28_security/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `security` (never `p28`) |
| Tables | `sec_*` |
| Soft delete | `is_deleted` |
| Cross-schema | UUID refs only |
| RLS | FORCE on tenant-scoped rows |
| Secrets | No key material columns |

---

## 2. Complete table inventory (**8 domain + 2 plumbing**)

| # | Table | Purpose |
|---|---|---|
| 1 | `sec_provider` | KMS/HSM/WAF provider registry |
| 2 | `sec_key` | Key metadata |
| 3 | `sec_key_version` | Versions / rotation |
| 4 | `sec_rotation_job` | Rotation orchestration |
| 5 | `sec_policy` | Security policy |
| 6 | `sec_posture_check` | Posture findings |
| 7 | `sec_waf_profile` | WAF integration contract |
| 8 | `sec_security_event` | Control-plane events |
| 9 | `sec_outbox` | Outbox |
| 10 | `sec_idempotency_key` | Idempotency |

---

## 3. Enumerations

| Enum | Values |
|---|---|
| Provider kind | `LOCAL_DEV`, `AWS_KMS`, `AZURE_KV`, `HSM`, `WAF` |
| Key purpose | `DATA`, `KEK`, `SIGN`, `WAF` |
| Key status | `ACTIVE`, `ROTATING`, `RETIRED` |
| Version state | `PENDING`, `ACTIVE`, `RETIRED` |
| Job status | `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED` |
| Posture result | `PASS`, `FAIL`, `UNKNOWN` |

---

## 4. Detailed tables

### 4.1 `sec_provider`

| Column | Type | Notes |
|---|---|---|
| `provider_key` | VARCHAR(80) UNIQUE | `LOCAL_DEV` |
| `kind` | VARCHAR(20) | |
| `status` | VARCHAR(30) | `ACTIVE` / `PROVIDER_PENDING` |
| `config_ref` | VARCHAR(200) NULL | p03 setting ref — not a secret |

### 4.2 `sec_key`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | NULL = system key |
| `key_alias` | VARCHAR(120) | Unique per tenant |
| `purpose` | VARCHAR(20) | |
| `algorithm` | VARCHAR(40) | e.g. `AES-256-GCM` |
| `provider_key` | VARCHAR(80) | |
| `current_version` | INTEGER | |
| `status` | VARCHAR(20) | |

**Partial unique:** `(tenant_id, key_alias)` where not deleted.

### 4.3 `sec_key_version`

| Column | Type | Notes |
|---|---|---|
| `key_id` | UUID | Same-schema ref |
| `version_no` | INTEGER | |
| `state` | VARCHAR(20) | |
| `wrapped_ref` | VARCHAR(500) | Opaque handle — **never raw key** |
| `activated_at` | TIMESTAMPTZ NULL | |
| `retired_at` | TIMESTAMPTZ NULL | |

**Unique:** `(key_id, version_no)`.

### 4.4 `sec_rotation_job`

| Column | Type | Notes |
|---|---|---|
| `key_id` | UUID | |
| `from_version` | INTEGER | |
| `to_version` | INTEGER | |
| `status` | VARCHAR(20) | |
| `error_code` | VARCHAR(80) NULL | |

### 4.5 `sec_policy`

| Column | Type | Notes |
|---|---|---|
| `policy_key` | VARCHAR(80) | |
| `version_no` | INTEGER | |
| `body` | JSONB | Controls |
| `is_active` | BOOLEAN | |

### 4.6 `sec_posture_check`

| Column | Type | Notes |
|---|---|---|
| `policy_id` | UUID | |
| `check_key` | VARCHAR(80) | |
| `result` | VARCHAR(20) | |
| `detail` | JSONB | |

### 4.7 `sec_waf_profile`

| Column | Type | Notes |
|---|---|---|
| `profile_key` | VARCHAR(80) | |
| `provider_key` | VARCHAR(80) | |
| `external_ref` | VARCHAR(200) | |
| `is_active` | BOOLEAN | |

### 4.8 `sec_security_event`

| Column | Type | Notes |
|---|---|---|
| `event_type` | VARCHAR(80) | |
| `payload` | JSONB | No secrets |
| `audit_ref` | UUID NULL | p19 event id |

### 4.9 Plumbing

`sec_outbox` (`stream` default `jesloterp:security:outbox`), `sec_idempotency_key` (`scope`, `key`, `response_json`).

---

## 5. RLS summary

| Table class | Policy |
|---|---|
| Tenant-scoped (`tenant_id` set) | `rls_bypass() OR tenant_id = current_tenant_id()` |
| System (`tenant_id` NULL) | Visible with bypass or admin session |
| Missing tenant GUC | Deny (fail-closed) |

---

## 6. Seed minimum

1. Provider `LOCAL_DEV` ACTIVE  
2. Providers `AWS_KMS`, `AZURE_KV`, `HSM` as `PROVIDER_PENDING`  
3. Permissions `security.*` …

---

## 7. ER overview

```text
sec_provider 1──* sec_key 1──* sec_key_version
                     │
                     └──* sec_rotation_job
sec_policy 1──* sec_posture_check
sec_waf_profile
sec_security_event
```

---

## 8. Implementation notes

No plaintext key columns. Alembic: `f28a0b1c2d3e` create + `f28b1c2d3e4f` RLS.
