# Extensibility Platform — Requirements Traceability Matrix

**Verification:** `pytest platforms/p29_extensibility/tests -q`  
**Companions:** `EXTENSIBILITY_GUIDE.md` · `EXTENSIBILITY_SCHEMA.md` · `EXTENSIBILITY_API.md` · `EXTENSIBILITY_IMPLEMENTATION_RECORD.md`

| Requirement ID | Source Document | Requirement | Backend Component/API | Implementation Status | Test Cases | Test Status |
| --- | --- | --- | --- | --- | --- | --- |
| EXT-G-01 | GUIDE §1 | Allow-listed hook pipeline | `PipelineService` | Implemented | bind + execute noop | PASS |
| EXT-G-02 | GUIDE §1 | No `eval`/`exec`/`compile` | builtin handlers + grep | Implemented | `test_no_eval` | PASS |
| EXT-G-03 | GUIDE §4 | Priority order | execute sort | Implemented | `test_handlers_run_in_priority_order` | PASS |
| EXT-G-04 | GUIDE §4 | FAIL_CLOSED stops later handlers | execute policy | Implemented | `test_fail_closed_stops_pipeline` | PASS |
| EXT-G-05 | GUIDE §4 | Timeout enforced per binding | `run_handler_isolated` + `EXT_TIMEOUT` | Implemented | `test_timeout_enforced_on_sleep_handler` | PASS |
| EXT-LIVE-01 | P29-LIVE-001 | Timeout kills isolated worker | spawn + `terminate`/`kill` | Implemented | `test_timeout_kills_isolated_worker` | PASS |
| EXT-G-08 | GUIDE §2 | Caller `execute` / `invoke_hooks` | `hooks.invoke_hooks` + p04/p05/p10 | Implemented | kernel points + metadata fail-closed | PASS |
| EXT-LIVE-02 | P29-LIVE-002 | More consumer commands | p04 update/delete, p05 after_publish, p10 cancel | Implemented | update/delete/cancel fail-closed + after_publish | PASS |
| EXT-G-06 | GUIDE §1 | No CRM-specific handlers | generic `resource_type` | Implemented | seed `kernel` only | PASS |
| EXT-G-07 | GUIDE §6 | Permissions `extensibility.*` | `require_ext_access` | Implemented | permission + unauth | PASS |
| EXT-S-01 | SCHEMA §2 | 6 domain + 2 plumbing | ORM catalog | Implemented | table count 8 | PASS |
| EXT-S-02 | SCHEMA RLS | FORCE RLS on executions | Alembic `f29b1c2d3e4f` | Implemented | RLS case `ext_execution` | PASS* |
| EXT-A-01 | API §6–7 | Bind / activate / execute | `/api/v1/extensibility/*` | Implemented | API contracts | PASS |
| EXT-A-02 | API errors | Unknown handler 422; not allow-listed 403 | `EXT_HANDLER_*` | Implemented | unknown + fail allow-list | PASS |
| EXT-A-03 | API | Inactive binding skipped | execute filter ACTIVE | Implemented | `test_inactive_binding_skipped` | PASS |
| EXT-M-01 | brief | ModulePlugin + internal router | `ExtensibilityModule` | Implemented | load_modules | PASS |
| EXT-M-02 | brief | Dual Alembic f29a / f29b | `alembic/versions/f29*` | Implemented | revision chain | PASS |

\* RLS integration skips when Postgres is unreachable. Timeout isolates allow-listed builtins in a child process and `terminate`/`kill`s it (`EXT_TIMEOUT`). Isolation is a process, not a container/seccomp sandbox.
