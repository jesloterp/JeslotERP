# Organization Platform — RTM

**Platform:** `p02_organization`  
**Last reviewed:** 2026-09-12  
**Test command:** `pytest platforms/p02_organization/tests tests/hygiene/test_hyg016_p02_schema_tables.py -q --tb=short`

| ID | Source | Requirement | Implementation | Status | Evidence |
|---|---|---|---|---|---|
| ORG-A-01 | API families | Tenant/company/branch HTTP | existing CQRS routers | Done | `test_org_api_family_contracts` |
| ORG-SOR-27a | AUD-011 | SQL out of HTTP routers | application queries/commands/services | Implemented | `test_context_tree_peel` |
| ORG-SOR-27b | AUD-016 | Document extra tables | SCHEMA extra-table list | Implemented | `ORGANIZATION_SCHEMA.md` |
| ORG-SOR-27c | TASK-SOR-027 | UoM + FX in p02 | `org_uom` / conversion / `org_fx_rate` | Implemented | `test_durable_measure` |
| ORG-HYG-016 | AUD-016 | Keep extras + column SCHEMA | 5 extras kept; lock every `__tablename__` | Implemented | `test_hyg016_p02_schema_tables` |

**Coverage note:** HTTP routers are thin. UoM/FX persist on `AsyncSession`; empty catalog is `[]`. Conversion refuses unknown pairs. No p34. Plant/SOrg encyclopedia not added.
