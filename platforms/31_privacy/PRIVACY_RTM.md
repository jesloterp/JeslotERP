# Privacy Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p31_privacy/tests -q`  
**Companions:** `PRIVACY_GUIDE.md` · `PRIVACY_SCHEMA.md` · `PRIVACY_API.md` · `PRIVACY_IMPLEMENTATION_RECORD.md`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| PRV-G-01 | GUIDE §1 | No blind domain DELETE | owner adapters; no SQL DELETE in p31 | Implemented | `test_p31_package_has_no_domain_delete_sql` | PASS |
| PRV-LIVE-01 | P31-LIVE-001 | Owning-platform erase adapters | p04 `SoftDeletePartnerHandler` + p01 `DeleteUserHandler` | Implemented | adapter delegate + HTTP SKIPPED | PASS |
| PRV-G-02 | GUIDE §2 | ACTIVE hold blocks erasure | DSR `BLOCKED` / `PRV_LEGAL_HOLD` | Implemented | `test_consent_withdraw_and_erasure_blocked_by_hold` | PASS |
| PRV-G-03 | GUIDE §2 | Consent withdraw auditable | withdraw + persist | Implemented | withdraw → WITHDRAWN | PASS |
| PRV-G-04 | GUIDE §2 | ACCESS DSR tenant-scoped | subject + crawlers | Implemented | `test_access_dsr_completes_without_delete` | PASS |
| PRV-LIVE-02 | P31-LIVE-002 | ACCESS index not empty | `PrivacyLocalCrawler` + p04/p01 locators | Implemented | local + partner crawler tests | PASS |
| PRV-G-05 | GUIDE §1 | p19 remains audit owner | events via outbox only | Implemented | no SIEM tables | PASS |
| PRV-G-06 | GUIDE §6 | Permissions `privacy.*` | `require_prv_access` | Implemented | auth matrix | PASS |
| PRV-S-01 | SCHEMA §2 | 7 domain + 2 plumbing | ORM catalog | Implemented | table count 9 | PASS |
| PRV-S-02 | SCHEMA RLS | FORCE RLS subjects | Alembic `f31b1c2d3e4f` | Implemented | RLS `prv_subject` | PASS* |
| PRV-A-01 | API §6–8 | Subject / consent / DSR | `/api/v1/privacy/*` | Implemented | API contracts | PASS |
| PRV-M-01 | brief | ModulePlugin + internal | `PrivacyModule` | Implemented | load_modules | PASS |
| PRV-M-02 | brief | Dual Alembic f31a / f31b | `alembic/versions/f31*` | Implemented | revision chain | PASS |

\* RLS integration skips when Postgres is unreachable.
