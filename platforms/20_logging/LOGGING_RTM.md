# Logging Platform — Requirements Traceability Matrix

**Verification:** `python -m pytest platforms/p20_logging/tests -q --tb=short` → **21 passed**

| Requirement ID | Source | Requirement | Component | Status | Tests | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| LOG-G-01 | GUIDE §1 | Structured ops logs ≠ audit | hot store + ingest | Implemented | ingest | PASS |
| LOG-G-02 | GUIDE §2 | Scrub before sink | `_scrub` + simulate | Implemented | phone `***` | PASS |
| LOG-G-03 | GUIDE §2 | DEBUG via time-boxed override | overrides + effective | Implemented | DEBUG then cancel | PASS |
| LOG-G-04 | GUIDE §3 | Fingerprints for ERROR+ | ingest | Implemented | fp ack/resolve | PASS |
| LOG-G-05 | GUIDE §6 | Permissions logging.* | HTTP gates | Implemented | 403 query | PASS |
| LOG-S-01 | SCHEMA §2 | 58 + plumbing = 60 | ORM | Implemented | table count 60 | PASS |
| LOG-A-01 | API §6 | Ingest batch + OTLP | public/internal | Implemented | ingest + otlp | PASS |
| LOG-A-02 | API §7 | Query window / request / trace / tail | query router | Implemented | TOO_BROAD + tail | PASS |
| LOG-A-03 | API §8 | Overrides + effective config | overrides | Implemented | conflict 409 | PASS |
| LOG-A-04 | API §9 | Fingerprints ack/resolve/hits | fingerprints | Implemented | 404 missing | PASS |
| LOG-A-05 | API §10 | Pipelines/scrub/sinks (no secrets) | pipeline | Implemented | simulate + 503 | PASS |
| LOG-A-06 | API §11–17 | Shippers, signals, stats, access, catalog, packs, health | ops | Implemented | shipper 404 + packs | PASS |
| LOG-MOD | registry | ModulePlugin deps p01 | LoggingModule + main/env | Implemented | load order | PASS |
| LOG-SOR-01 | TASK-SOR-018 | Hot ingest/query → Postgres; empty `[]` | `logging_repository` + `require_logging_access` | Implemented | empty + persist-then-fetch | PASS |
| LOG-SOR-02 | TASK-SOR-018 | Sink port; no fake live vendor | `SinkPort` + factory | Implemented | STDOUT ACTIVE; OTLP PENDING | PASS |
