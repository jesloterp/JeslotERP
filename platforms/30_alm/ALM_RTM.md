# ALM Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p30_alm/tests -q`  
**Companions:** `ALM_GUIDE.md` · `ALM_SCHEMA.md` · `ALM_API.md` · `ALM_IMPLEMENTATION_RECORD.md`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| ALM-G-01 | GUIDE §1 | Landscape transport, not Git | `AlmService` (no git APIs) | Implemented | seal/export/import only | PASS |
| ALM-G-02 | GUIDE §3 | SHA256 seal + checksum | `_checksum` + seal | Implemented | `test_checksum_mismatch` | PASS |
| ALM-G-03 | GUIDE §3 | Missing dependency blocks import | `ALM_DEPENDENCY_MISSING` | Implemented | `test_missing_dependency_blocks_import` | PASS |
| ALM-G-04 | GUIDE §3 | PROD promote requires signature | `ALM_SIGNATURE_REQUIRED` | Implemented | `test_prod_promote_requires_signature` | PASS |
| ALM-LIVE-01 | P30-LIVE-001 | HSM sign + multi-env deploy; no HMAC stub | p28 `HsmAdapter.sign` + `HttpDeployAgent` | Implemented | sign pending + HMAC rejected + agent pending | PASS |
| ALM-G-05 | GUIDE §4 | Idempotent same-version import | import existing → `idempotent` | Implemented | `test_seal_export_import_idempotent` | PASS |
| ALM-G-06 | GUIDE §4 | Rollback stores previous version | `rollback` | Implemented | `test_rollback_and_auth` | PASS |
| ALM-G-07 | GUIDE §6 | Permissions `alm.*` | `require_alm_access` | Implemented | auth matrix | PASS |
| ALM-S-01 | SCHEMA §2 | 8 domain + 2 plumbing | ORM (`package_version` col) | Implemented | table count 10 | PASS |
| ALM-S-02 | SCHEMA RLS | FORCE RLS overlay packages | Alembic `f30b1c2d3e4f` | Implemented | RLS `alm_package` | PASS* |
| ALM-A-01 | API §6–8 | Seal/export/import/promote/rollback | `/api/v1/alm/*` | Implemented | API contracts | PASS |
| ALM-M-01 | brief | ModulePlugin + internal | `AlmModule` | Implemented | load_modules | PASS |
| ALM-M-02 | brief | Dual Alembic f30a / f30b | `alembic/versions/f30*` | Implemented | revision chain | PASS |

\* RLS integration skips when Postgres is unreachable.
