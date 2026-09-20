# JeslotERP ALM Platform — Production Schema (Advanced)

**Version:** 1.0 (Advanced / Enterprise)  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — lean SoR (AUD-022); Alembic `f30a0b1c2d3e` / `f30b1c2d3e4f`  
**Package:** `platforms.p30_alm`  
**PostgreSQL schema:** `alm`  
**Companion:** [`ALM_GUIDE.md`](ALM_GUIDE.md) · [`ALM_API.md`](ALM_API.md)

> Runtime models: `platforms/p30_alm/infrastructure/persistence/models/`.

---

## 1. Conventions

| Topic | Rule |
|---|---|
| Schema | `alm` (never `p30`) |
| Tables | `alm_*` |
| Cross-schema | Artifact refs are UUID + kind strings |
| RLS | FORCE on tenant overlay packages |

---

## 2. Complete table inventory (**8 domain + 2 plumbing**)

| # | Table | Purpose |
|---|---|---|
| 1 | `alm_environment` | DEV / QA / PROD |
| 2 | `alm_package` | Transport unit |
| 3 | `alm_artifact` | Component in package |
| 4 | `alm_manifest` | Deps + checksum |
| 5 | `alm_signature` | Integrity signature |
| 6 | `alm_import_run` | Import attempts |
| 7 | `alm_promotion` | Landscape move |
| 8 | `alm_rollback` | Prior version pointer |
| 9 | `alm_outbox` | Outbox |
| 10 | `alm_idempotency_key` | Idempotency |

---

## 3. Enumerations

| Enum | Values |
|---|---|
| Env code | `DEV`, `QA`, `PROD` |
| Layer | `SYSTEM_CORE`, `TENANT_OVERLAY` |
| Package status | `DRAFT`, `SEALED`, `EXPORTED`, `IMPORTED`, `PROMOTED`, `ROLLED_BACK` |
| Artifact kind | `METADATA`, `CONFIG_REF`, `RULE`, `PROCESS`, `OTHER` |

---

## 4. Detailed tables

### 4.1 `alm_environment`

| Column | Type | Notes |
|---|---|---|
| `code` | VARCHAR(10) UNIQUE | DEV/QA/PROD |
| `name` | VARCHAR(80) | |

### 4.2 `alm_package`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID NULL | Overlay vs core |
| `package_key` | VARCHAR(120) | |
| `version` | VARCHAR(40) | Semver-ish |
| `status` | VARCHAR(20) | |
| `layer` | VARCHAR(20) | |

**Partial unique:** `(tenant_id, package_key, version)`.

### 4.3 `alm_artifact`

| Column | Type | Notes |
|---|---|---|
| `package_id` | UUID | |
| `kind` | VARCHAR(30) | |
| `artifact_ref` | UUID | Foreign platform id — no FK |
| `layer` | VARCHAR(20) | |

### 4.4 `alm_manifest`

| Column | Type | Notes |
|---|---|---|
| `package_id` | UUID UNIQUE | |
| `depends_on` | JSONB | `[{package_key, min_version}]` |
| `checksum_sha256` | VARCHAR(64) | |

### 4.5 `alm_signature`

| Column | Type | Notes |
|---|---|---|
| `package_id` | UUID | |
| `algorithm` | VARCHAR(40) | `HSM-PKCS11` (local HMAC is not accepted) |
| `signature` | VARCHAR(500) | |

### 4.6 `alm_import_run`

| Column | Type | Notes |
|---|---|---|
| `package_id` | UUID | |
| `environment_code` | VARCHAR(10) | |
| `status` | VARCHAR(20) | |
| `idempotency_key` | VARCHAR(80) | |

### 4.7 `alm_promotion`

| Column | Type | Notes |
|---|---|---|
| `package_id` | UUID | |
| `from_env` | VARCHAR(10) | |
| `to_env` | VARCHAR(10) | |
| `status` | VARCHAR(20) | |

### 4.8 `alm_rollback`

| Column | Type | Notes |
|---|---|---|
| `package_id` | UUID | |
| `previous_version` | VARCHAR(40) | |
| `reason` | VARCHAR(200) | |

---

## 5. RLS summary

Tenant overlay packages/runs: FORCE RLS. System core: admin/bypass.

---

## 6. Seed minimum

Environments DEV, QA, PROD. Permissions `alm.*`.

---

## 7. ER overview

```text
alm_environment
alm_package 1──* alm_artifact
     │ 1──1 alm_manifest
     │ 1──* alm_signature
     └──* alm_import_run / alm_promotion / alm_rollback
```

---

## 8. Implementation notes

Alembic `f30a0b1c2d3e` / `f30b1c2d3e4f`.
