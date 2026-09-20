# Sharing Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p33_sharing/tests -q`  
**Companions:** `SHARING_GUIDE.md` · `SHARING_SCHEMA.md` · `SHARING_API.md` · `SHARING_IMPLEMENTATION_RECORD.md`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| SHR-G-01 | GUIDE §1 | Record ACL ≠ tenant RLS | `SharingService.evaluate` | Implemented | owner vs stranger | PASS |
| SHR-G-02 | GUIDE §4 | OWNER → MANUAL → TEAM → RULE → HIERARCHY | evaluate chain | Implemented | owner / team / hierarchy | PASS |
| SHR-G-03 | GUIDE §2 | Duplicate grant idempotent | unique grant key | Implemented | `test_idempotent_grant` | PASS |
| SHR-G-04 | GUIDE §4 | Stranger denied | evaluate deny | Implemented | stranger `allowed` false | PASS |
| SHR-G-05 | GUIDE §1 | Generic resource UUID only | `resource_type` + UUID | Implemented | `kernel.record` | PASS |
| SHR-G-06 | GUIDE §6 | Permissions `sharing.*` | `require_shr_access` | Implemented | auth matrix | PASS |
| SHR-S-01 | SCHEMA §2 | 6 domain + 2 plumbing | ORM catalog | Implemented | table count 8 | PASS |
| SHR-S-02 | SCHEMA RLS | FORCE RLS grants | Alembic `f33b1c2d3e4f` | Implemented | RLS `shr_grant` | PASS* |
| SHR-A-01 | API §6–8 | Teams / grants / evaluate | `/api/v1/sharing/*` | Implemented | API contracts | PASS |
| SHR-M-01 | brief | ModulePlugin + internal | `SharingModule` | Implemented | load_modules | PASS |
| SHR-M-02 | brief | Dual Alembic f33a / f33b | `alembic/versions/f33*` | Implemented | revision chain | PASS |
| SHR-G-07 | GUIDE §2 | p04 GET evaluates when grants exist | `assert_record_access` + `assert_partner_share` | Implemented | skip / owner / stranger | PASS |
| SHR-S-03 | GUIDE SoR | List grants DB-first on AsyncSession | `fetch_grants` | Implemented | `test_fetch_grants_skips_mock_session` | PASS |
| SHR-LIVE-01 | P33-LIVE-001 | Evaluate reads grants from Postgres, not process ledger only | `SharingService.evaluate` + `fetch_grants` | Implemented | `test_evaluate_uses_fetched_grants_not_memory_only` | PASS |
| SHR-LIVE-02 | P33-LIVE-002 | Share-filter p04 list partners | `filter_visible_partners` + `can_access_record` | Implemented | skip open / owner keep / stranger drop | PASS |
| SHR-LIVE-03 | P33-LIVE-003 | More p04 domain reads evaluate (not CRM) | `require_partner_share` | Implemented | public partner-id GET + validate-for-use | PASS |
| SHR-LIVE-04 | K28-33-001 | List teams DB-first on AsyncSession | `fetch_teams` | Implemented | `test_list_teams_db_first_empty_not_memory` | PASS |

\* RLS integration skips when Postgres is unreachable.
