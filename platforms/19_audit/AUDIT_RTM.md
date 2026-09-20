# Audit Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p19_audit/tests -q` → **25 passed**

| Requirement ID | Source | Requirement | Component | Status | Tests | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| AUD-G-01 | GUIDE §1 | Append-only events; no update/delete APIs | ingest-only HTTP | Implemented | ingest tests | PASS |
| AUD-G-02 | GUIDE §2 | Hash chain prev_hash + event_hash (SHA-256) | `catalog_store.ingest_event` / verify | Implemented | verify + chain-break | PASS |
| AUD-G-03 | GUIDE §2 | Secrets never in before/after | field mask + never_capture | Implemented | password `***` | PASS |
| AUD-G-04 | GUIDE §5 | Ingest explicit + event-bus binding | public + `/ingest/from-event` | Implemented | internal from-event | PASS |
| AUD-G-05 | GUIDE §3 | Legal hold blocks purge | `start_purge` | Implemented | hold/purge 409 then release | PASS |
| AUD-G-06 | GUIDE §6 | Permissions audit.* | HTTP gates | Implemented | 403 denied query | PASS |
| AUD-G-07 | GUIDE §3 | Audit-of-audit on reads | `aud_access_event` / `/access` | Implemented | access list | PASS |
| AUD-S-01 | SCHEMA §2 | 60 + outbox + idempotency = 62 | ORM | Implemented | `test_aud_module_tables_count_62` | PASS |
| AUD-S-02 | SCHEMA §15 | Seed actions, object types, retention, masks, writers | `seed_defaults` | Implemented | catalog + retention | PASS |
| AUD-A-01 | API §6 | POST events; idempotent source_event_id replay 200 | ingest | Implemented | replay meta | PASS |
| AUD-A-02 | API §6 | Batch + from-event | internal | Implemented | batch errors + binding | PASS |
| AUD-A-03 | API §7 | Query requires window or object_id | `/query` | Implemented | TOO_BROAD + scoped | PASS |
| AUD-A-04 | API §7 | Get / field-changes / timeline | query router | Implemented | get + 404 | PASS |
| AUD-A-05 | API §7 | Break-glass + sensitive | BG token | Implemented | 403 then 200 | PASS |
| AUD-A-06 | API §8 | Streams, verify, seals | integrity | Implemented | ok + CHAIN_BROKEN | PASS |
| AUD-A-07 | API §9 | Retention, holds, purge | ops | Implemented | hold/purge/retention | PASS |
| AUD-A-08 | API §10 | Export case / package / download-url | ops | Implemented | NOT_READY then READY | PASS |
| AUD-A-09 | API §11 | Alerts + SIEM + tick | ops + internal | Implemented | fire/ack/deliver | PASS |
| AUD-SOR-17 | TASK-SOR-017 | Append-only event SoR | durable_ingest_event + fetch_events | Implemented | `test_durable_sor` | PASS |
| AUD-SOR-24 | TASK-SOR-024 | SIEM HTTP port fail-closed (AUD-019) | HttpxSiemForwarder + adapter_factory | Implemented | `test_siem_forwarder` + test-connection | PASS |
| AUD-HYG-019 | AUD-019 | Same as TASK-SOR-024; stay in p19 | pointer + hygiene lock | Implemented | `test_hyg019_siem_http` | PASS |
| AUD-A-10 | API §12–15 | Catalog, access, saved, packs, health | ops | Implemented | catalog + packs | PASS |
| AUD-MOD | registry | ModulePlugin deps p01+p02+p13 | AuditModule + main/env | Implemented | load order | PASS |
