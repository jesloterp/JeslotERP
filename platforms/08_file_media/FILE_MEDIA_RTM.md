# JeslotERP File Media Platform (p08) — Requirements Traceability Matrix

**Date:** 2026-09-10  
**Package:** `platforms.p08_file_media`  
**PostgreSQL schema:** `media`  
**Sources:** `FILE_MEDIA_GUIDE.md`, `FILE_MEDIA_SCHEMA.md`, `FILE_MEDIA_API.md`  
**Verification:** `pytest platforms/p08_file_media/tests -q` → **81 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| FM-G-01 | GUIDE §1 | Binary content control plane (not local /uploads) | `p08_file_media` ModulePlugin + signed storage | Done | `test_media_module_*` | Pass |
| FM-G-02 | GUIDE §2 | No cross-schema FKs; UUID refs | ORM models UUID columns only | Done | `test_media_all_65_tables_registered` | Pass |
| FM-G-03 | GUIDE §3 | Status machine INIT→…→AVAILABLE/QUARANTINED | `MediaCatalogStore` + enums | Done | upload/scan tests | Pass |
| FM-G-04 | GUIDE §4 | Direct signed / multipart / proxy flows | uploads router + local storage | Done | `test_media_upload_*` | Pass |
| FM-G-05 | GUIDE §5 | Storage backends LOCAL_FS + S3_COMPAT seeds | catalog seed `local_dev`/`s3_primary` | Done | startup seeds | Pass |
| FM-G-06 | GUIDE §6 | AV scan + quarantine isolation | scan callback + quarantine APIs | Done | `test_media_scan_*` | Pass |
| FM-G-07 | GUIDE §6 | Content policy ext/MIME/size | `_validate_content` | Done | deny MIME/ext tests | Pass |
| FM-G-08 | GUIDE §6 | Permissions `media.*` | `permissions/catalog.py` | Done | permissions gate tests | Pass |
| FM-G-09 | GUIDE §7 | Quota reserve/finalize/release | quota reservation in store | Done | quota + abort tests | Pass |
| FM-G-10 | GUIDE §8 | Legal hold blocks delete/purge | legal-hold APIs | Done | legal hold tests | Pass |
| FM-G-11 | GUIDE §9 | Lifecycle HOT/COOL/ARCHIVE + restore | tier/restore APIs | Done | tier/restore tests | Pass |
| FM-G-12 | GUIDE §11 | Module layout services/http/storage | `platforms/p08_file_media/**` | Done | module tests | Pass |
| FM-G-13 | GUIDE §12 | Outbox stream `jesloterp:media:outbox` | `MediaOutboxEvent` + store emit | Done | health + stream test | Pass |
| FM-S-01 | SCHEMA §2 | 62 domain tables | ORM split by section | Done | 65-table assert | Pass |
| FM-S-02 | SCHEMA §2 | Plumbing outbox/idempotency/catalog_audit | 3 plumbing tables | Done | 65-table assert | Pass |
| FM-S-03 | SCHEMA §1 | Schema name `media` never `p08` | `MEDIA_SCHEMA` | Done | schema constant test | Pass |
| FM-S-04 | SCHEMA §3 | Enumerations status/scan/tier/job/share | `domain/enums.py` | Done | runtime flows | Pass |
| FM-S-05 | SCHEMA §15 | Seeds backends/classifications/policy/variants/purposes/quota | `seed_defaults` | Done | startup seeds | Pass |
| FM-A-01 | API §5 | Permission codes | MEDIA_PERMISSIONS | Done | catalog test | Pass |
| FM-A-02 | API §6 | Upload init/multipart/complete/abort/parts/proxy | `/api/v1/media/uploads/*` | Done | upload flow tests | Pass |
| FM-A-03 | API §7 | Object get/list/patch/delete/undelete/purge | objects router | Done | object/delete tests | Pass |
| FM-A-04 | API §8 | Download/preview URL rules AVAILABLE only | download-url | Done | download/scan tests | Pass |
| FM-A-05 | API §9 | Variants reprocess + job poll | variants + processing-jobs | Done | variant tests | Pass |
| FM-A-06 | API §10 | Scan/quarantine/review | quarantine APIs | Done | quarantine tests | Pass |
| FM-A-07 | API §11 | Attachments + collections | attachments router | Done | attachment tests | Pass |
| FM-A-08 | API §12 | Shares redeem/revoke + ACL | share APIs | Done | share tests | Pass |
| FM-A-09 | API §13 | Legal hold / retention / tier / restore | objects/admin routers | Done | hold/tier tests | Pass |
| FM-A-10 | API §14 | Quotas/me + policies | admin quotas | Done | quota tests | Pass |
| FM-A-11 | API §15 | Backends/policies/variants/packages | admin router | Done | admin contracts | Pass |
| FM-A-12 | API §16 | CDN purge | cdn endpoints | Done | cdn tests | Pass |
| FM-A-13 | API §17 | Audit downloads/catalog | audit endpoints | Done | audit tests | Pass |
| FM-A-14 | API §18 | Internal worker APIs | `/internal/v1/media/*` | Done | internal tests | Pass |
| FM-A-15 | API §4 | Error codes envelope | `MediaError` handlers | Done | 422/403/409/410 paths | Pass |
| FM-A-16 | API §1 | Idempotent init/complete | Idempotency-Key | Done | idempotent tests | Pass |
| FM-A-17 | API | Magic-byte / checksum mismatch | magic_sniff + complete | Done | magic/checksum tests | Pass |
| FM-A-18 | API | Proxy capped by feature flag | `media.proxy_upload` | Done | proxy tests | Pass |
| FM-W-01 | sample.md | RTM 100% | this file | Done | — | Pass |
| FM-W-02 | sample.md | Implementation record 18 sections | `FILE_MEDIA_IMPLEMENTATION_RECORD.md` | Done | — | Pass |
| FM-W-03 | FILE_MEDIA wiring | Alembic DDL + FORCE RLS + main.py ModuleRegistry | `e3f4a5b6c7d8` / `f4a5b6c7d8e9` + apps/api/main.py | Implemented | test_load_modules_includes_file_media | Passed |
| FM-W-04 | alembic/env.py | Import p08 persistence models | alembic/env.py | Implemented | alembic heads includes f4a5… | Passed |
| FM-SOR-10 | TASK-SOR-010 | Blob metadata SoR; S3 port fail-closed | media_repository + PendingS3StorageAdapter | Implemented | `test_durable_sor` + `test_storage_port` | PASS |
| FM-SOR-23 | TASK-SOR-023 | Virus-scan adapter fail-closed (AUD-018) | ClamAvScanEngine + adapter_factory | Implemented | `test_scan_port` + scan-engine test-connection | PASS |
| FM-HYG-018 | AUD-018 | Same as TASK-SOR-023; stay in p08 | pointer + hygiene lock | Implemented | `test_hyg018_virus_scanner` | PASS |

**Coverage notes**

- HTTP object/quarantine persist on `AsyncSession`; empty list is `[]`. `MediaCatalogStore` is the TestClient double. S3 is `PROVIDER_PENDING`.
- Live ClamAV is a port: pytest stays MEMORY; `ClamAvScanEngine` returns `ERROR` / `PROVIDER_PENDING` (no invented CLEAN).
- Parent wiring completed: ModuleRegistry + exception handlers + Alembic create/RLS.
