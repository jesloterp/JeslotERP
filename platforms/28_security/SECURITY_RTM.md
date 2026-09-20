# Security Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p28_security/tests -q` → **16 passed**  
**Companions:** `SECURITY_GUIDE.md` · `SECURITY_SCHEMA.md` · `SECURITY_API.md` · `SECURITY_IMPLEMENTATION_RECORD.md`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| SEC-G-01 | GUIDE §1 | KMS/policy/posture/WAF/events only | `SecurityService` + schema `security` | Implemented | module tables + API | PASS |
| SEC-G-02 | GUIDE §1 | No p01/p03/p19 duplication | permissions + GET strip | Implemented | `test_create_and_get_key_has_no_secret_fields` | PASS |
| SEC-G-03 | GUIDE §1 | Never return key material | `get_key` / list strip `wrapped_ref` | Implemented | no-secret GET | PASS |
| SEC-G-04 | GUIDE §4 | Rotation concurrency lock | `rotate_key` in-flight | Implemented | `test_rotate_and_in_flight` | PASS |
| SEC-G-05 | GUIDE §2 | Provider ports LOCAL_DEV / pending cloud | `KmsPort` + `get_kms` | Implemented | `test_kms_port` | PASS |
| SEC-LIVE-01 | P28-LIVE-001 | AWS/Azure/HSM never invent wrap; pytest never hits vendor | named adapters + test-connection | Implemented | `test_kms_port` + API pending | PASS |
| SEC-LIVE-02 | P28-LIVE-002 | Public GET never `_secret_plain` / wrapped_ref | `_public_key` | Implemented | planted-plain GET | PASS |
| SEC-G-06 | GUIDE §6 | Permissions `security.*` | `require_sec_access` | Implemented | `test_permission_denied_and_unauthenticated` | PASS |
| SEC-G-07 | GUIDE §10 | Persist when session is `AsyncSession` | `persist_row` / `fetch_keys` | Implemented | `test_durable_sor` | PASS |
| SEC-S-01 | SCHEMA §1 | Schema `security` never `p28` | `SECURITY_SCHEMA` | Implemented | module constant | PASS |
| SEC-S-02 | SCHEMA §2 | 8 domain + 2 plumbing | ORM catalog | Implemented | table count 10 | PASS |
| SEC-S-03 | SCHEMA §5 | FORCE RLS tenant rows | Alembic `f28b1c2d3e4f` | Implemented | RLS case `sec_security_event` | PASS* |
| SEC-A-01 | API §4–6 | Keys create/list/get/rotate/retire | `/api/v1/security/keys*` | Implemented | API contracts | PASS |
| SEC-A-02 | API §4 | `SEC_PROVIDER_UNKNOWN` / `SEC_ROTATION_IN_FLIGHT` | `SecurityError` | Implemented | unknown provider + rotate | PASS |
| SEC-A-03 | API §5 / §9 | Unauth 401 / no perm 403 | `require_sec_access` | Implemented | permissions gate | PASS |
| SEC-A-04 | API §1 | Thin router → `SecurityService` | public router | Implemented | module + contracts | PASS |
| SEC-M-01 | brief | ModulePlugin + `/internal/v1/security` | `SecurityModule` | Implemented | load_modules | PASS |
| SEC-M-02 | brief | Dual Alembic f28a / f28b | `alembic/versions/f28*` | Implemented | revision chain | PASS |

\* RLS integration skips when Postgres is unreachable.
