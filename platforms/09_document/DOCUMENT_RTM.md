# JeslotERP Document Platform (p09) — Requirements Traceability Matrix

**Date:** 2026-09-10  
**Package:** `platforms.p09_document`  
**PostgreSQL schema:** `document`  
**Sources:** `DOCUMENT_GUIDE.md`, `DOCUMENT_SCHEMA.md`, `DOCUMENT_API.md`  
**Verification:** `pytest platforms/p09_document/tests -q` → **82 passed**

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| DOC-G-01 | GUIDE §1 | DMS control plane; bytes in p08 | `p09_document` + media_id refs | Done | create/media tests | Pass |
| DOC-G-02 | GUIDE §2 | Number via p07; content via p08 | stub gateways in-process | Done | allocate + media tests | Pass |
| DOC-G-03 | GUIDE §2 | No cross-schema FKs | UUID columns only | Done | ORM models | Pass |
| DOC-G-04 | GUIDE §3 | DIR-first + type-driven policy | `DocumentCatalogStore.create_document` | Done | create variations | Pass |
| DOC-G-05 | GUIDE §3 | Status network enforced | transitions + CONTRACT edges | Done | status success/deny | Pass |
| DOC-G-06 | GUIDE §3 | Version immutability on RELEASED | `is_immutable` + DocImmutableError | Done | immutable tests | Pass |
| DOC-G-07 | GUIDE §3 | Check-out exclusivity | checkout lock | Done | conflict + break-lock | Pass |
| DOC-G-08 | GUIDE §3 | Object links many-to-many | links + by-entity | Done | link/list/conflict | Pass |
| DOC-G-09 | GUIDE §3 | Templates + merge allow-list | generate + MergeFieldInvalid | Done | generate success/invalid | Pass |
| DOC-G-10 | GUIDE §3 | Rendition pipeline async | request + internal callback | Done | rendition/callback | Pass |
| DOC-G-11 | GUIDE §3 | ACL layered fail-closed | put_acl + get deny | Done | ACL deny/grant | Pass |
| DOC-G-12 | GUIDE §3 | Legal hold blocks delete/obsolete | legal hold services | Done | hold delete/obsolete | Pass |
| DOC-G-13 | GUIDE §3 | Compound documents | put/validate-release | Done | incomplete/success | Pass |
| DOC-G-14 | GUIDE §3 | Packages install | checksum verify | Done | checksum fail/success | Pass |
| DOC-G-15 | GUIDE §6 | Permissions `document.*` | permissions/catalog.py | Done | permissions gate | Pass |
| DOC-G-16 | GUIDE §7 | Module layout services/http/gateways | `platforms/p09_document/**` | Done | module tests | Pass |
| DOC-G-17 | GUIDE §8 | Outbox `jesloterp:document:outbox` | DocOutboxEvent + store emit | Done | stream + health | Pass |
| DOC-S-01 | SCHEMA §2 | 64 domain tables | ORM split by section | Done | 68-table assert | Pass |
| DOC-S-02 | SCHEMA §2 | Plumbing outbox/idempotency/catalog_audit/status_history | 4 plumbing tables | Done | 68-table assert | Pass |
| DOC-S-03 | SCHEMA §1 | Schema name `document` never `p09` | `DOCUMENT_SCHEMA` | Done | schema constant test | Pass |
| DOC-S-04 | SCHEMA §3 | Enumerations lifecycle/checkout/link/esign/… | `domain/enums.py` | Done | runtime flows | Pass |
| DOC-S-05 | SCHEMA §15 | Seeds types/status/links/series/templates/renditions | `seed_defaults` | Done | startup seeds | Pass |
| DOC-A-01 | API §5 | Permission codes | DOCUMENT_PERMISSIONS | Done | catalog + gate tests | Pass |
| DOC-A-02 | API §6 | DIR create/get/list/patch/delete/by-number | `/api/v1/documents` | Done | API create/get | Pass |
| DOC-A-03 | API §7 | Versions list/add/download/compare | versions routes | Done | checkin/compare/download | Pass |
| DOC-A-04 | API §8 | Checkout/checkin/undo/break | checkout routes | Done | exclusivity + break | Pass |
| DOC-A-05 | API §9 | Status transitions + sugar | transitions/release/obsolete | Done | network API tests | Pass |
| DOC-A-06 | API §10 | Object links + by-entity | links + by-entity | Done | API link tests | Pass |
| DOC-A-07 | API §11 | Libraries/folders/move | libraries routes | Done | type allow/deny | Pass |
| DOC-A-08 | API §12 | Templates + generate | templates/generate | Done | generate API | Pass |
| DOC-A-09 | API §13 | Renditions + distribute | rendition/distribute | Done | distribute API | Pass |
| DOC-A-10 | API §14 | ACL + shares | acl/shares | Done | ACL API | Pass |
| DOC-A-11 | API §15 | Comments + review tasks | comments/review-tasks | Done | comments unit | Pass |
| DOC-A-12 | API §16 | Legal hold + retention | legal-holds/retention | Done | hold API | Pass |
| DOC-A-13 | API §17 | E-sign envelopes + webhooks | esign + internal webhook | Done | esign API/callback | Pass |
| DOC-A-14 | API §18 | Compound validate-release | compound routes | Done | compound API | Pass |
| DOC-A-15 | API §19 | Types/policy/network/packages | admin catalog | Done | types/packages API | Pass |
| DOC-A-16 | API §20 | Audit access/catalog | audit routes | Done | permissions audit deny | Pass |
| DOC-A-17 | API §21 | Internal callbacks + health | `/internal/v1/documents/*` | Done | internal health/callbacks | Pass |
| DOC-A-18 | API §4 | Error codes envelope | `DocumentError` handlers | Done | 403/409/422 paths | Pass |
| DOC-A-19 | API §1 | Idempotent create/generate/check-in | Idempotency-Key | Done | idempotent create | Pass |
| DOC-A-20 | API §4 | Media not available / quarantined | gateway checks | Done | media API tests | Pass |
| DOC-W-01 | sample.md | RTM 100% | this file | Done | — | Pass |
| DOC-W-02 | sample.md | Implementation record 18 sections | `DOCUMENT_IMPLEMENTATION_RECORD.md` | Done | — | Pass |
| DOC-W-03 | DOCUMENT wiring | `apps/api/main.py` ModuleRegistry + exception handlers | apps/api/main.py | Implemented | test_load_modules_includes_document | Passed |
| DOC-W-04 | DOCUMENT_SCHEMA DDL/RLS | Alembic create + FORCE RLS | `a5b6c7d8e9f0` / `b6c7d8e9f0a1` | Implemented | alembic upgrade head | Passed |
| DOC-W-05 | alembic/env.py | Import p09 persistence models | alembic/env.py | Implemented | alembic heads | Passed |
| DOC-W-06 | Registry | PLATFORM_REGISTRY Live for p09 | PLATFORM_REGISTRY.md | Implemented | registry Live + checkbox | Passed |
| DOC-SOR-11 | TASK-SOR-011 | DMS HTTP → Postgres (not ArchiveLink) | document_repository + library_repository | Implemented | `test_durable_sor` | PASS |

---

**Coverage note:** All GUIDE/SCHEMA/API functional requirements mapped. ModuleRegistry + Alembic wiring completed and verified (75 tests; DB head `b6c7d8e9f0a1`).
