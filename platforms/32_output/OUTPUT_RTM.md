# Output Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p32_output/tests -q`  
**Companions:** `OUTPUT_GUIDE.md` · `OUTPUT_SCHEMA.md` · `OUTPUT_API.md` · `OUTPUT_IMPLEMENTATION_RECORD.md`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| OUT-G-01 | GUIDE §1 | Not p08/p09/p15 | determine + render only | Implemented | no SMTP / invoice types | PASS |
| OUT-G-02 | GUIDE §3 | TEXT renderer live | `TextRenderer` | Implemented | determine + Hello A | PASS |
| OUT-G-03 | GUIDE §3 | PDF provider-pending | `WeasyPrintPdfAdapter` + `OUT_PDF_PROVIDER_PENDING` | Implemented | PDF 422 + ping | PASS |
| OUT-LIVE-01 | P32-LIVE-001 | Live PDF port + p15 hand-off; no SMTP | adapter + `handoff_to_p15` | Implemented | pending + EMAIL ACCEPTED + no smtplib | PASS |
| OUT-G-04 | GUIDE §4 | Determination locale + channel | `determine` NOTICE/en/PRINT | Implemented | `test_determine_and_text_render` | PASS |
| OUT-G-05 | GUIDE §4 | Retry increments attempts | `retry` | Implemented | attempts == 2 | PASS |
| OUT-G-06 | GUIDE §6 | Permissions `output.*` | `require_out_access` | Implemented | auth matrix | PASS |
| OUT-S-01 | SCHEMA §2 | 6 domain + 2 plumbing | ORM catalog | Implemented | table count 8 | PASS |
| OUT-S-02 | SCHEMA RLS | FORCE RLS jobs | Alembic `f32b1c2d3e4f` | Implemented | RLS `out_job` | PASS* |
| OUT-A-01 | API §6–7 | Templates / determine / render / retry | `/api/v1/output/*` | Implemented | API contracts | PASS |
| OUT-M-01 | brief | ModulePlugin + internal | `OutputModule` | Implemented | load_modules | PASS |
| OUT-M-02 | brief | Dual Alembic f32a / f32b | `alembic/versions/f32*` | Implemented | revision chain | PASS |

\* RLS integration skips when Postgres is unreachable.
