# JeslotERP Organization Platform — Developer Integration Guide

**Version:** 1.3  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — HTTP SQL peeled; extras kept in SCHEMA (HYG-016); UoM + FX in p02. Not Production.  
**Primary stack:** Python 3.12+, FastAPI, SQLAlchemy 2.x, PostgreSQL, Alembic  
**Bounded Context:** Organization / Enterprise (ORG)  
**Identity owner:** IAM  
**Document owner:** JeslotERP Platform Engineering

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial production architecture baseline. |
| 1.1 | 2026-09-08 | Soft-delete + partial-unique guidance (§11); per-context schema made mandatory and reconciled with IAM (§4); Rule 9 refined to allow guarded polymorphic satellites while keeping FK-first for real relations (§36, Rule 9); native-enum evolution warning (§33); added Appendix E (PII & retention) and Appendix F (identifier/code generation). |
| 1.2 | 2026-09-12 | TASK-SOR-027: SQL out of HTTP routers (AUD-011); extra tables documented (AUD-016); UoM + FX stay in p02 (no p34). |
| 1.3 | 2026-09-12 | HYG-016: five extra ORM tables kept with columns in SCHEMA (not migrated). Lock: `test_hyg016_p02_schema_tables`. |

---

## 1. Purpose

The Organization Platform is the authoritative bounded context for organizational and enterprise master data in JeslotERP.

It owns:

- Tenants
- Companies / legal entities
- Branches
- Employees
- Departments
- Designations
- Warehouses
- Warehouse locations
- Cost centers
- Profit centers
- Fiscal years
- Financial periods
- Fiscal-period closures
- Working calendars
- Holidays
- Tax registrations
- Addresses
- Contacts
- Global geographic/reference masters
- Tenant settings, branding, subscription metadata and deployment metadata
- Units of measure and conversions (platform UoM catalog — **not p34**)
- Exchange rates and conversion (platform FX catalog — **not p34**)

HTTP routers are thin adapters. SQLAlchemy `select` / raw SQL live in application queries, commands, and repositories. Health `SELECT 1` is the only probe left in a router.

It does **not** own:

- User authentication
- Passwords
- Sessions
- Access tokens
- Refresh tokens
- Roles
- Permissions
- OAuth/OIDC clients
- MFA
- SSO identities

Those belong to the IAM bounded context.

---

# 2. Architectural Position

```text
                         ┌──────────────────────┐
                         │       IAM             │
                         │                      │
                         │ Users                │
                         │ Roles                │
                         │ Permissions          │
                         │ Sessions             │
                         │ Tokens               │
                         └──────────┬───────────┘
                                    │
                              user_id reference
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                         ORG PLATFORM                           │
│                                                               │
│ Tenant → Company → Branch                                    │
│              │          │                                    │
│              │          ├── Warehouse → Location             │
│              │          └── Employees                         │
│              ├── Departments                                  │
│              ├── Designations                                 │
│              ├── Cost Centers                                 │
│              ├── Profit Centers                               │
│              ├── Fiscal Years → Financial Periods             │
│              └── Tax Registrations                             │
└───────────────────────────┬───────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
       FINANCE             SALES           INVENTORY
```

The ORG platform is a **master-data authority**. Other modules consume organization IDs; they should not duplicate organizational ownership rules.

---

# 3. Core Ownership Rules

## 3.1 IAM owns identity

IAM is authoritative for:

```text
user_id
email
phone
password
MFA
session
OAuth
roles
permissions
```

ORG may store `user_id` as a reference for an employee, but ORG must not implement authentication.

## 3.2 ORG owns organizational structure

ORG is authoritative for:

```text
tenant_id
company_id
branch_id
employee_id
department_id
designation_id
warehouse_id
location_id
cost_center_id
profit_center_id
fiscal_year_id
financial_period_id
tax_registration_id
```

## 3.3 Business modules own business transactions

Finance owns:

```text
accounts
journal entries
invoices
payments
receipts
ledger
tax transactions
```

Sales owns:

```text
quotations
sales orders
customers
sales invoices
```

Inventory owns:

```text
items
stock
stock movements
inventory valuation
```

Never make ORG the owner of module-specific transactional data.

---

# 4. Database Strategy

Recommended PostgreSQL schema:

```text
org
iam
finance
sales
inventory
procurement
crm
```

Each bounded context **must** own its own PostgreSQL schema namespace, even inside a single database. This is the platform standard, not a suggestion — IAM already ships on `schema="iam"`/`"identity"`, and ORG must match it.

Example (required form):

```python
class OrgCompany(Base):
    __tablename__ = "org_company"
    __table_args__ = {"schema": "org"}
```

> **Consistency note:** The current `org` model set uses a flat namespace with `org_*` table prefixes and no `schema=`. That is drift from this standard and from IAM. Treat migrating ORG to `schema="org"` as required cleanup (one Alembic revision: `CREATE SCHEMA org; ALTER TABLE ... SET SCHEMA org;`), scheduled before the platform is split. The `org_*` prefix stays regardless, so table references remain unambiguous during the transition.

---

# 5. Tenant Hierarchy

The fundamental hierarchy is:

```text
Tenant
  └── Company
       └── Branch
            ├── Employee
            └── Warehouse
                 └── Location
```

Additional company-level structures:

```text
Company
 ├── Department
 ├── Designation
 ├── Cost Center
 ├── Profit Center
 ├── Fiscal Year
 ├── Working Calendar
 └── Tax Registration
```

Every application service must respect this hierarchy.

---

# 6. Tenant Isolation

Tenant isolation is a security boundary.

Every tenant-owned record must be scoped by:

```text
tenant_id
```

Company-scoped records additionally use:

```text
company_id
```

Branch-scoped records additionally use:

```text
branch_id
```

Do not trust IDs supplied independently by the client.

The authenticated request context must come from the IAM-issued token.

Example:

```python
@dataclass(frozen=True)
class RequestContext:
    user_id: UUID
    tenant_id: UUID | None
    company_id: UUID | None
    branch_id: UUID | None
    session_id: UUID | None
    scopes: frozenset[str]
```

---

# 7. Context Security

A normal ERP request should look like:

```text
JWT
 │
 ├── sub = user_id
 ├── tenant_id
 ├── company_id
 └── branch_id
       │
       ▼
RequestContext
       │
       ▼
ORG authorization / hierarchy validation
       │
       ▼
Business module
```

Do not use:

```text
X-Tenant-ID
X-Company-ID
X-Branch-ID
```

as authoritative security controls.

Headers can be used for tracing or non-security hints, but never to override the authenticated context.

---

# 8. Context Switching

A user may belong to multiple tenants, companies or branches.

When the user changes context:

```text
Current JWT
   │
   ▼
POST /context/switch
   │
   ├── validate user assignment in IAM
   ├── validate tenant
   ├── validate company belongs to tenant
   ├── validate branch belongs to company
   ├── resolve permissions
   └── issue new access token
```

Never mutate an existing JWT.

Example target:

```json
{
  "tenant_id": "tenant-b",
  "company_id": "company-2",
  "branch_id": "branch-4"
}
```

The newly issued token contains the new context.

---

# 9. ID and FK Rules

Use PostgreSQL UUID for all entity identifiers.

```python
id: Mapped[UUID] = mapped_column(
    UUID(as_uuid=True),
    primary_key=True,
    default=uuid.uuid4,
    server_default=text("gen_random_uuid()"),
)
```

Avoid mixing:

```text
UUID
CHAR(32)
VARCHAR IDs
integer IDs
```

for the same conceptual identifier.

---

# 10. Hierarchical Referential Integrity

Application validation is required, but database constraints should provide defense in depth.

Example:

```text
org_company
UNIQUE(id, tenant_id)
```

Then:

```text
org_branch
FOREIGN KEY(company_id, tenant_id)
REFERENCES org_company(id, tenant_id)
```

This prevents:

```text
tenant A
 └── branch
      └── company from tenant B
```

The same pattern should be applied to:

- Employee → Company
- Employee → Branch
- Employee → Department
- Employee → Designation
- Warehouse → Company
- Warehouse → Branch
- Location → Warehouse
- Fiscal Period → Fiscal Year
- Holiday → Calendar
- Tax Registration → Company / Branch

---

# 11. Unique Constraint Rules

Do not automatically make business codes globally unique.

Preferred:

```text
UNIQUE(tenant_id, company_id, code)
```

Examples:

```text
Department code
Designation code
Warehouse code
Cost center code
Profit center code
Employee code
```

Company code is generally:

```text
UNIQUE(tenant_id, code)
```

Tenant code may be globally unique if the platform requires it.

### 11.1 Soft delete breaks plain UNIQUE constraints

Because master records are soft-deleted (§13), a **plain** `UNIQUE(tenant_id, code)` keeps counting deleted rows — so once code `ACME` is soft-deleted, it can never be reused. This is almost never the intended behavior.

Enforce natural-key uniqueness with a **partial unique index scoped to live rows**:

```python
# instead of UniqueConstraint("tenant_id", "code", name="uq_org_company_tenant_code")
Index(
    "uq_org_company_tenant_code",
    "tenant_id", "code",
    unique=True,
    postgresql_where=text("is_deleted = false"),
)
```

```sql
CREATE UNIQUE INDEX uq_org_company_tenant_code
    ON org.org_company (tenant_id, code)
    WHERE is_deleted = false;
```

Apply the same `WHERE is_deleted = false` pattern to every business/natural key (codes, numbers, names). **Exception:** immutable-id hierarchy keys used as composite-FK targets — e.g. `UNIQUE(id, tenant_id)` — stay as full `UNIQUE` constraints, since a foreign key cannot reference a partial index and `id` never collides regardless of delete state.

"One current / one primary / one default" rules follow the same idea with a richer predicate:

```sql
CREATE UNIQUE INDEX uq_org_contact_primary
    ON org.org_contact (tenant_id, owner_type, owner_id)
    WHERE is_primary = true AND is_deleted = false;
```

---

# 12. Base Model Strategy

Use different base classes instead of putting every field on every table.

Recommended:

```python
PlatformBase
TenantBase
CompanyBase
BranchBase
```

Conceptually:

```text
PlatformBase
 ├── global masters

TenantBase
 ├── tenant-owned entities

CompanyBase
 ├── company-owned entities

BranchBase
 └── branch-owned entities
```

Common cross-cutting mixins:

```text
UUIDMixin
AuditMixin
LifecycleMixin
VersionMixin
ExtensionMixin
```

---

# 13. Lifecycle and Soft Delete

Do not physically delete important ERP master records casually.

Use:

```text
status
is_deleted
deleted_at
deleted_by
```

Recommended lifecycle:

```text
ACTIVE
INACTIVE
SUSPENDED
DELETED
```

Deletion rules must be business-specific.

For example:

```text
Company with posted financial transactions
    → cannot be physically deleted
```

Instead:

```text
status = INACTIVE
```

---

# 14. Optimistic Concurrency

Entities that can be concurrently edited should use versioning.

Example:

```text
version
row_version
```

An update should verify the version originally read by the client.

Conceptually:

```sql
UPDATE org_company
SET name = :name,
    version = version + 1
WHERE id = :id
  AND version = :expected_version;
```

If zero rows are updated:

```text
OptimisticConcurrencyError
```

Return HTTP 409 at the API boundary.

---

# 15. Organization API Layers

Recommended structure:

```text
org/
├── domain/
│   ├── entities/
│   ├── value_objects/
│   ├── services/
│   ├── events/
│   └── repositories/
│
├── application/
│   ├── commands/
│   ├── queries/
│   ├── handlers/
│   ├── dto/
│   └── ports/
│
├── infrastructure/
│   ├── persistence/
│   │   ├── models/
│   │   ├── repositories/
│   │   └── unit_of_work/
│   ├── cache/
│   ├── messaging/
│   └── external/
│
└── presentation/
    └── api/
        ├── routers/
        ├── schemas/
        ├── dependencies/
        └── exception_handlers/
```

---

# 16. Domain Layer Rules

Domain code must not import:

```text
FastAPI
SQLAlchemy
Pydantic
Redis
Celery
JWT libraries
HTTP clients
PostgreSQL drivers
```

Domain entities should express business rules.

Example:

```python
@dataclass
class Company:
    id: UUID
    tenant_id: UUID
    code: str
    name: str

    def deactivate(self) -> None:
        if self.status == RecordStatus.DELETED:
            raise DomainError("Deleted company cannot be deactivated")

        self.status = RecordStatus.INACTIVE
```

---

# 17. Application Layer

Application services coordinate use cases.

Example:

```python
class CreateCompanyHandler:
    def __init__(
        self,
        company_repository: CompanyRepository,
        tenant_gateway: TenantGateway,
        uow: UnitOfWork,
    ):
        self.company_repository = company_repository
        self.tenant_gateway = tenant_gateway
        self.uow = uow

    async def handle(
        self,
        command: CreateCompanyCommand,
        context: RequestContext,
    ) -> UUID:
        await self.tenant_gateway.ensure_access(
            context.user_id,
            command.tenant_id,
        )

        company = Company.create(
            tenant_id=command.tenant_id,
            code=command.code,
            name=command.name,
        )

        await self.company_repository.add(company)
        await self.uow.commit()

        return company.id
```

---

# 18. Repository Ports

Define repository interfaces in the domain/application boundary.

Example:

```python
class CompanyRepository(Protocol):

    async def get(
        self,
        company_id: UUID,
        tenant_id: UUID,
    ) -> Company | None:
        ...

    async def add(self, company: Company) -> None:
        ...

    async def update(self, company: Company) -> None:
        ...

    async def exists_by_code(
        self,
        tenant_id: UUID,
        code: str,
    ) -> bool:
        ...
```

Never expose SQLAlchemy models to the domain.

---

# 19. Query Architecture

CQRS is recommended.

Commands:

```text
CreateCompany
UpdateCompany
DeactivateCompany
CreateBranch
AssignEmployee
CreateWarehouse
CloseFinancialPeriod
```

Queries:

```text
GetCompany
ListCompanies
GetBranch
ListBranches
GetEmployee
ListEmployees
GetCurrentFiscalPeriod
GetOrganizationTree
```

Query handlers may use optimized SQLAlchemy Core queries.

Do not force every read through aggregate loading.

---

# 20. FastAPI Integration

FastAPI should only translate:

```text
HTTP request
    ↓
Pydantic request model
    ↓
Application command/query
    ↓
handler
    ↓
domain
    ↓
repository
```

Example:

```python
@router.post("/companies")
async def create_company(
    request: CreateCompanyRequest,
    context: RequestContext = Depends(get_request_context),
    handler: CreateCompanyHandler = Depends(get_create_company_handler),
):
    company_id = await handler.handle(
        CreateCompanyCommand(
            tenant_id=context.tenant_id,
            code=request.code,
            name=request.name,
        ),
        context,
    )

    return {"id": str(company_id)}
```

---

# 21. Authorization

ORG should not own the global permission system.

Use an IAM authorization port:

```python
class AuthorizationPort(Protocol):

    async def authorize(
        self,
        context: RequestContext,
        permission: str,
    ) -> None:
        ...
```

Example:

```python
await authorization.authorize(
    context,
    "org.company.create",
)
```

Then perform organization-specific validation.

Authorization has two layers:

```text
IAM permission
      +
ORG hierarchy/access validation
```

Both must succeed.

---

# 22. Organization Gateway

Other bounded contexts should not directly query ORG tables.

Expose an integration contract:

```python
class OrganizationGateway(Protocol):

    async def get_company(
        self,
        tenant_id: UUID,
        company_id: UUID,
    ) -> CompanyReference | None:
        ...

    async def get_branch(
        self,
        tenant_id: UUID,
        company_id: UUID,
        branch_id: UUID,
    ) -> BranchReference | None:
        ...

    async def validate_hierarchy(
        self,
        tenant_id: UUID,
        company_id: UUID | None,
        branch_id: UUID | None,
    ) -> None:
        ...
```

---

# 23. Modular Monolith vs Microservices

Initially, prefer:

```text
IAM + ORG + Finance + Sales
```

inside one deployable application if operational simplicity is important.

Keep boundaries at code/database-schema level.

Later:

```text
auth.jesloterp.com
org.jesloterp.com
finance.jesloterp.com
sales.jesloterp.com
```

can be separated.

The application should already use ports/gateways so extraction does not require rewriting business logic.

---

# 24. Event-Driven Integration

Important ORG events:

```text
TenantCreated
TenantDeactivated

CompanyCreated
CompanyUpdated
CompanyDeactivated

BranchCreated
BranchUpdated
BranchDeactivated

EmployeeCreated
EmployeeUpdated
EmployeeDeactivated

WarehouseCreated
WarehouseUpdated

FiscalYearOpened
FinancialPeriodOpened
FinancialPeriodClosed
FinancialPeriodReopened

TaxRegistrationCreated
TaxRegistrationUpdated
```

Events should contain identifiers and relevant facts, not secrets.

Example:

```json
{
  "event_id": "uuid",
  "event_type": "org.company.created",
  "event_version": 1,
  "occurred_at": "2026-09-08T10:00:00Z",
  "tenant_id": "uuid",
  "company_id": "uuid",
  "data": {
    "code": "COMP001",
    "name": "Example Pvt Ltd"
  }
}
```

---

# 25. Transactional Outbox

When a command changes database state:

```text
BEGIN
  UPDATE org_company
  INSERT outbox_event
COMMIT
```

Then:

```text
Outbox Worker
     ↓
Message Broker
     ↓
Finance / Sales / Inventory
```

Never:

```text
UPDATE database
COMMIT
publish event
```

without an outbox.

Otherwise a crash between commit and publish can lose the event.

---

# 26. Idempotency

Consumers must be idempotent.

Store:

```text
event_id
consumer_name
processed_at
```

or an equivalent inbox mechanism.

If:

```text
org.company.created
```

is delivered twice, Finance must not create duplicate dependent records.

---

# 27. Cache Strategy

Redis may cache:

```text
tenant metadata
company metadata
branch metadata
organization tree
timezone
currency
permission-independent reference data
```

Redis is never the source of truth.

Pattern:

```text
PostgreSQL = source of truth
Redis       = acceleration layer
```

Invalidate cache after successful database commit.

---

# 28. API Versioning

Use:

```text
/api/v1/org/...
```

Example:

```text
GET    /api/v1/org/tenants
GET    /api/v1/org/companies
POST   /api/v1/org/companies
GET    /api/v1/org/companies/{company_id}
PATCH  /api/v1/org/companies/{company_id}

GET    /api/v1/org/branches
POST   /api/v1/org/branches

GET    /api/v1/org/employees
POST   /api/v1/org/employees
```

Never expose internal database table names as the public API contract.

---

# 29. API Error Contract

Use a consistent response:

```json
{
  "error": {
    "code": "ORG_COMPANY_NOT_FOUND",
    "message": "Company was not found.",
    "details": {},
    "request_id": "uuid"
  }
}
```

Recommended HTTP mappings:

```text
400 validation error
401 unauthenticated
403 authorization failure
404 resource not found
409 conflict / optimistic concurrency
422 semantic validation
429 rate limit
500 unexpected error
```

Do not expose SQLAlchemy/PostgreSQL exception messages to clients.

---

# 30. Pagination

List APIs should support:

```text
page
page_size
sort
search
status
```

For large tables prefer cursor/keyset pagination.

Example:

```text
GET /api/v1/org/employees?
    cursor=...
    &limit=50
```

Avoid unrestricted:

```text
SELECT * FROM org_employee
```

---

# 31. Filtering

Filtering must always be tenant-scoped.

Bad:

```python
select(OrgCompany).where(OrgCompany.id == company_id)
```

Better:

```python
select(OrgCompany).where(
    OrgCompany.id == company_id,
    OrgCompany.tenant_id == context.tenant_id,
)
```

For company-scoped resources:

```python
.where(
    OrgEmployee.tenant_id == context.tenant_id,
    OrgEmployee.company_id == context.company_id,
)
```

Never rely on the client to provide the correct tenant filter.

---

# 32. PostgreSQL RLS

For high-assurance tenant isolation, PostgreSQL Row-Level Security can provide defense in depth.

Application authorization remains mandatory.

Conceptually:

```sql
ALTER TABLE org.org_company ENABLE ROW LEVEL SECURITY;
```

Set transaction-local context:

```sql
SET LOCAL app.tenant_id = '...';
```

Policy:

```sql
CREATE POLICY tenant_isolation
ON org.org_company
USING (
    tenant_id = current_setting('app.tenant_id')::uuid
);
```

RLS should be introduced deliberately and tested thoroughly.

---

# 33. Database Migrations

Use Alembic.

Rules:

```text
One logical migration = one revision
Never edit an already-applied migration
Review generated migrations manually
Test upgrade
Test downgrade where supported
Run migrations before application rollout
```

### 33.1 Native PostgreSQL enums evolve awkwardly

Native enum types (`SAEnum(RecordStatus, name="record_status")`, `tax_registration_type`, etc.) are compact and self-documenting, but changing them is migration-heavy:

- Adding a value needs `ALTER TYPE ... ADD VALUE` — which **cannot run inside a transaction block** on older PostgreSQL, so autogenerated Alembic revisions often need `op.execute(...)` outside the transaction.
- Removing or renaming a value effectively requires recreating the type and rewriting every dependent column.

For enumerations expected to change with business needs (types, categories), prefer a `String` + `CheckConstraint`, or a lookup table (e.g. the existing `ConfigSystemCode`). Reserve native enums for **stable, code-controlled** sets (lifecycle status, deployment mode). Autogenerated enum migrations must always be reviewed by hand.

Production pipeline:

```text
Build
 ↓
Migration validation
 ↓
Backup
 ↓
Migration
 ↓
Application rollout
 ↓
Health check
```

---

# 34. SQLAlchemy Session Rules

Use one request-scoped async session.

Recommended:

```python
async with async_sessionmaker() as session:
    async with session.begin():
        ...
```

Application code should use a Unit of Work abstraction.

Do not create random sessions deep inside domain/application services.

---

# 35. Transaction Boundaries

A command handler should normally own one transaction:

```text
HTTP request
    ↓
Command
    ↓
Handler
    ↓
UoW BEGIN
    ↓
Domain changes
    ↓
Outbox event
    ↓
COMMIT
```

Do not commit inside individual repository methods.

Bad:

```python
await repository.add(company)
await repository.commit()
await repository.add(branch)
await repository.commit()
```

Better:

```python
await repository.add(company)
await repository.add(branch)
await uow.commit()
```

---

# 36. Address and Contact Design

The governing principle is **FK-first**: when strong referential integrity is required, do not use an untyped polymorphic `(reference_table, reference_id)` pair. Choose one of two enforceable patterns instead.

**Pattern A — shared entity + typed composite FK (preferred for addresses).**
Keep one `org_address` table and give each owner a direct, tenant-safe FK. This is what the ORG schema does today:

```text
org_address            UNIQUE(id, tenant_id)

org_company.address_id
   FOREIGN KEY (address_id, tenant_id)
   REFERENCES org_address(id, tenant_id)
```

The composite FK guarantees an address can never cross tenants, and PostgreSQL enforces the reference. Use this for company / branch / warehouse / tenant addresses.

**Pattern B — explicit relation tables.**
When an owner can have many of something with its own attributes, use per-owner link tables:

```text
org_company_address
org_branch_address
org_employee_address
```

### 36.1 Guarded polymorphic satellites (narrow exception)

For **lightweight satellite data with no downstream FK dependents** — `org_contact` is the canonical case — a *guarded* polymorphic owner is acceptable and avoids table sprawl. It is permitted only with all of these guardrails:

```text
owner_type   NOT NULL, constrained by CHECK to a closed set
owner_id     NOT NULL
CHECK (owner_type IN ('COMPANY','BRANCH','WAREHOUSE','TENANT'))
INDEX (tenant_id, owner_type, owner_id)
integrity of (owner_type, owner_id) enforced at the service layer
```

Do **not** use a polymorphic owner for anything another table must FK to, for financial/tax records, or for hierarchy edges — those must use Pattern A or B. In short: FK-first everywhere; guarded polymorphic only for leaf satellites like contacts.

---

# 37. Tax Registration

Do not assume one GSTIN per company.

Use:

```text
Company
 └── Tax Registrations
       ├── GSTIN
       ├── State
       ├── Registration Type
       ├── Effective Dates
       └── Status
```

Finance/GST modules should reference:

```text
tax_registration_id
```

rather than copying GSTIN everywhere.

---

# 38. Fiscal Year and Financial Period

Hierarchy:

```text
Company
  └── Fiscal Year
        └── Financial Period
```

Recommended lifecycle:

```text
OPEN
SOFT_CLOSED
CLOSED
```

Reopening a closed period must be an explicit privileged operation and audited.

Never silently change historical financial state.

---

# 39. Warehouse Hierarchy

```text
Company
 └── Branch
      └── Warehouse
           └── Location
                └── Parent Location
```

Location hierarchy can represent:

```text
Warehouse
 ├── Zone A
 │    ├── A-01
 │    └── A-02
 └── Zone B
      ├── B-01
      └── B-02
```

Inventory owns stock quantities; ORG owns physical organizational warehouse structure.

---

# 40. Employee Integration with IAM

ORG employee:

```text
employee.user_id
```

references an IAM user conceptually.

IAM user:

```text
user_id
```

does not become an ORG employee automatically.

Possible states:

```text
IAM User
   │
   ├── no employee
   │
   └── Employee
```

This allows users such as:

```text
tenant owner
external accountant
auditor
API service user
```

without forcing every IAM identity to be an employee.

---

# 41. No Cross-Bounded-Context ORM Relationships

Avoid:

```python
from iam.models import IAMUser
```

inside ORG ORM models.

Instead:

```python
user_id: Mapped[UUID | None]
```

and resolve identity through an integration port when necessary.

This prevents IAM and ORG from becoming one tightly coupled module.

---

# 42. Security Rules

Never log:

```text
password
OTP
refresh token
access token
client secret
MFA secret
database credentials
connection strings
session secrets
```

Use:

```text
structured logging
request_id
trace_id
tenant_id
user_id
operation
duration
result
```

Only log identifiers that are safe and necessary.

---

# 43. Audit Requirements

Audit sensitive organization actions:

```text
company.created
company.updated
company.deactivated

branch.created
branch.updated

employee.created
employee.updated
employee.deactivated

tax_registration.created
tax_registration.updated

fiscal_period.closed
fiscal_period.reopened

tenant.settings.updated
tenant.branding.updated
```

Audit records should include:

```text
event_id
tenant_id
user_id
action
resource_type
resource_id
timestamp
request_id
IP/device information where policy permits
before/after summary where appropriate
```

Do not store secrets in audit payloads.

---

# 44. API Authorization Matrix

Example:

| Operation | Permission |
|---|---|
| Create tenant | `org.tenant.create` |
| Update tenant | `org.tenant.update` |
| Create company | `org.company.create` |
| Update company | `org.company.update` |
| Create branch | `org.branch.create` |
| Update branch | `org.branch.update` |
| Create employee | `org.employee.create` |
| Update employee | `org.employee.update` |
| Create warehouse | `org.warehouse.create` |
| Close period | `org.finance.period.close` |
| Reopen period | `org.finance.period.reopen` |
| Manage tax registration | `org.tax_registration.manage` |

IAM decides whether the caller has the permission.

ORG verifies whether the requested resource belongs to the caller's current organizational context.

---

# 45. Service-to-Service Integration

For internal services:

```text
Finance → ORG
Sales → ORG
Inventory → ORG
```

Use authenticated service identity.

Recommended:

```text
OAuth2 client credentials
```

or equivalent internal service authentication.

Never trust:

```text
X-Service-Name: finance
```

as authentication.

---

# 46. Integration DTOs

Do not expose ORM objects to other modules.

Use stable reference DTOs:

```python
@dataclass(frozen=True)
class CompanyReference:
    id: UUID
    tenant_id: UUID
    code: str
    name: str
    status: str
```

This protects downstream modules from database refactoring.

---

# 47. Organization Tree API

Provide an optimized read model for UI:

```text
Tenant
 ├── Company A
 │    ├── Branch A1
 │    ├── Branch A2
 │    └── Warehouse
 │
 └── Company B
      └── Branch B1
```

This should be a query/read model, not a reason to load every SQLAlchemy relationship recursively.

---

# 48. Testing Strategy

Minimum testing layers:

```text
Unit
Integration
API
Authorization
Tenant-isolation
Concurrency
Migration
Contract
End-to-end
```

Critical security tests:

```text
User from tenant A cannot read tenant B company
User from company A cannot read company B
User from branch A cannot modify branch B
Branch from company B cannot be assigned to company A
Employee department cannot belong to another company
Warehouse branch cannot belong to another company
Fiscal period cannot belong to another fiscal year
Tax registration cannot cross companies
```

---

# 49. Adversarial Tests

Every protected endpoint should be tested with modified IDs.

Example:

```text
Valid token:
tenant=A
company=A1
branch=A1B1

Request:
company_id=A1
branch_id=B1
```

Expected:

```text
403 or 404
```

Never:

```text
200
```

Test both:

```text
wrong tenant ID
wrong company ID
wrong branch ID
wrong employee ID
wrong warehouse ID
```

---

# 50. Performance Guidelines

Required indexes should generally cover:

```text
tenant_id
tenant_id + company_id
tenant_id + company_id + branch_id
status
code
foreign keys
created_at
```

Do not add indexes blindly.

Measure with:

```sql
EXPLAIN (ANALYZE, BUFFERS)
```

for important production queries.

---

# 51. Background Jobs

Celery or another worker may be used for:

```text
organization cache rebuild
large imports
bulk employee imports
holiday synchronization
reference master synchronization
event processing
```

Never perform long-running jobs inside HTTP requests.

---

# 52. Bulk Import

For CSV/Excel imports:

```text
Upload
 ↓
Validate file
 ↓
Create import job
 ↓
Background worker
 ↓
Validate rows
 ↓
Transaction batches
 ↓
Error report
 ↓
Completion event
```

Never trust uploaded organization IDs without tenant/context validation.

---

# 53. API Idempotency

For mutation endpoints that may be retried:

```text
POST /companies
POST /branches
POST /employees
```

support:

```text
Idempotency-Key
```

where appropriate.

Store:

```text
tenant_id
user_id
idempotency_key
request_hash
response
created_at
```

Do not allow the same key to execute a materially different request.

---

# 54. Observability

Every request should have:

```text
request_id
trace_id
tenant_id
user_id
operation
status
latency
```

Metrics:

```text
org_api_requests_total
org_api_request_duration
org_db_query_duration
org_event_publish_failures
org_outbox_pending
org_cache_hit_ratio
```

---

# 55. Health Checks

Provide:

```text
/health/live
/health/ready
```

Liveness:

```text
application process is alive
```

Readiness:

```text
PostgreSQL reachable
required dependencies available
```

Do not expose secrets or detailed infrastructure information in public health endpoints.

---

# 56. Deployment

Recommended:

```text
Docker
PostgreSQL
Redis
Celery
Nginx / Load Balancer
Secrets Manager
Observability
```

Production deployment:

```text
Git
 ↓
CI
 ↓
Tests
 ↓
Security scan
 ↓
Build immutable image
 ↓
Migration
 ↓
Deploy
 ↓
Health check
```

---

# 57. Configuration

Use environment/configuration management.

Example:

```text
APP_ENV
DATABASE_URL
REDIS_URL
IAM_ISSUER
IAM_AUDIENCE
ORG_EVENT_BUS
LOG_LEVEL
```

Never commit secrets.

Use secret references for:

```text
database credentials
JWT signing material
OAuth client secrets
external API credentials
```

---

# 58. Recommended Dependency Direction

```text
presentation
     ↓
application
     ↓
domain

infrastructure ─────→ domain/application ports
```

Never:

```text
domain → infrastructure
domain → FastAPI
domain → SQLAlchemy
```

---

# 59. Definition of Done

An ORG feature is not complete until:

- [ ] Domain rules implemented
- [ ] Application command/query implemented
- [ ] Repository port implemented
- [ ] SQLAlchemy adapter implemented
- [ ] Migration created
- [ ] Tenant isolation enforced
- [ ] Hierarchy validation implemented
- [ ] Authorization implemented
- [ ] API schema implemented
- [ ] Error contract implemented
- [ ] Audit requirement reviewed
- [ ] Domain events reviewed
- [ ] Outbox behavior reviewed
- [ ] Unit tests added
- [ ] Integration tests added
- [ ] Cross-tenant security tests added
- [ ] Concurrency behavior tested
- [ ] API documentation updated
- [ ] Observability added where required

**TASK-SOR-027 (this slice):**
- [x] SQL out of HTTP routers (application services / queries / commands)
- [x] Extra ORM tables documented vs SCHEMA (AUD-016)
- [x] UoM + FX tables + convert in p02 (no p34)
- [x] Empty measure catalog on `AsyncSession` is `[]`
- [x] RLS GUC on `require_org_access`; `org_fx_rate` tenant policy

**HYG-016 (this slice):**
- [x] Five extras kept (not migrated): bank, system code, onboarding, outbox, idempotency
- [x] Column-level SCHEMA + lock that every `__tablename__` is named

---

# 60. Final Engineering Rules

These rules are mandatory for this platform.

### Rule 1
IAM owns identity and authorization.

### Rule 2
ORG owns organizational structure.

### Rule 3
Finance/Sales/Inventory own their own transactions.

### Rule 4
Never trust tenant/company/branch IDs from the client.

### Rule 5
JWT context is the authoritative request context.

### Rule 6
Context switching issues a new token.

### Rule 7
Database constraints must provide defense in depth for tenant hierarchy.

### Rule 8
Never store plaintext database connection strings or credentials in tenant master data.

### Rule 9
Never use polymorphic IDs where strong FK integrity is required. Use a typed composite FK to a shared entity, or explicit relation tables (§36). A *guarded* polymorphic owner is allowed only for leaf satellites with no FK dependents, such as `org_contact` (§36.1).

### Rule 10
Never allow bounded contexts to directly import each other's ORM models.

### Rule 11
Commands change state; queries optimize reads.

### Rule 12
Database state and integration events must be committed atomically through an outbox.

### Rule 13
Consumers must be idempotent.

### Rule 14
Redis is a cache, not the source of truth.

### Rule 15
Every security-sensitive operation must be auditable.

### Rule 16
Every tenant-scoped query must enforce tenant context.

### Rule 17
Production schema changes must go through reviewed Alembic migrations.

### Rule 18
Never expose database implementation details through public APIs.

### Rule 19
Prefer explicit domain contracts over shared ORM models.

### Rule 20
Design ORG so it can later be extracted into an independent service without rewriting its domain.

---

# 61. Recommended Package Naming

```text
jesloterp/
└── modules/
    └── organization/
        ├── domain/
        ├── application/
        ├── infrastructure/
        └── presentation/
```

Suggested module names:

```text
organization
tenant
company
branch
employee
department
designation
warehouse
location
cost_center
profit_center
fiscal
calendar
tax_registration
address
contact
```

Keep the public module boundary:

```python
jesloterp.modules.organization
```

Other modules should depend on integration ports/contracts rather than internal implementation paths.

---

# 62. Integration Contract Summary

The stable contract between ORG and other JeslotERP modules is:

```text
INPUT
-----
RequestContext
tenant_id
company_id
branch_id
resource IDs


ORG PROVIDES
------------
TenantReference
CompanyReference
BranchReference
EmployeeReference
WarehouseReference
FiscalPeriodReference
TaxRegistrationReference
OrganizationGateway


ORG GUARANTEES
--------------
Tenant hierarchy
Company hierarchy
Branch hierarchy
Organizational status
Reference-data consistency
Authorization-context validation
Domain events
```

Other modules should treat these IDs as foreign references owned by ORG.

---

# 63. Target Architecture

The final target is:

```text
                         JESLOTERP PLATFORM
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
         IAM                    ORG                 PLATFORM
          │                      │                      │
       Identity             Organization          Billing/SaaS
       Access               Master Data           Infrastructure
       Security                  │
          │                      │
          └──────────────┬───────┘
                         │
                  Authenticated
                  Request Context
                         │
        ┌────────────────┼─────────────────┐
        ▼                ▼                 ▼
     Finance           Sales           Inventory
        │                │                 │
        └────────────────┼─────────────────┘
                         │
                  Event / Outbox
                         │
                    Integration
```

This architecture allows JeslotERP to begin as a modular monolith and later extract IAM, ORG or business modules into independent services without changing the domain contracts.

---

## Appendix A — Canonical Request Flow

```text
1. Client sends access token
2. API validates signature/issuer/audience/expiry
3. API extracts user + organizational context
4. RequestContext is created
5. IAM authorization checks permission
6. ORG validates hierarchy/resource ownership
7. Application handler executes use case
8. Repository changes PostgreSQL state
9. Outbox event is written in same transaction
10. Transaction commits
11. Worker publishes event
12. Consumers process event idempotently
13. Response returned with request_id
```

---

## Appendix B — Canonical Resource Validation

For a request containing:

```text
tenant_id
company_id
branch_id
```

validate:

```text
1. User has access to tenant
2. Tenant exists and is active
3. Company exists
4. Company belongs to tenant
5. Company is active
6. Branch exists if supplied
7. Branch belongs to company
8. Branch is active
9. User has required permission
10. Target resource belongs to the same hierarchy
```

A failure at any stage must stop the operation.

---

## Appendix C — What Must Never Happen

```text
❌ Finance imports OrgCompany SQLAlchemy model
❌ ORG imports IAM User ORM model
❌ Client controls tenant_id using a header
❌ Company from tenant A referenced by branch from tenant B
❌ Plaintext database password stored in org_tenant
❌ Raw refresh token stored
❌ Domain imports FastAPI
❌ Repository commits independently
❌ Event published without transactional outbox
❌ Duplicate event creates duplicate business data
❌ Closed financial period silently reopened
❌ Global company codes accidentally conflict across tenants
❌ API exposes SQLAlchemy exceptions
❌ Redis becomes the source of truth
```

---

## Appendix D — Production Checklist

Before production:

```text
[ ] PostgreSQL production configuration
[ ] UUID generation extension available
[ ] Alembic migrations reviewed
[ ] Composite FK constraints tested
[ ] Tenant isolation tests passing
[ ] Authorization tests passing
[ ] RLS decision documented
[ ] Backup and restore tested
[ ] Connection pooling configured
[ ] Redis configured
[ ] Outbox worker configured
[ ] Dead-letter/retry strategy configured
[ ] Structured logging configured
[ ] Metrics configured
[ ] Tracing configured
[ ] Secrets manager configured
[ ] API rate limits configured
[ ] Audit logging enabled
[ ] Disaster recovery procedure documented
[ ] Migration rollback/recovery plan documented
[ ] Load tests completed
[ ] Security review completed
```

---

## Appendix E — Data Privacy, PII & Retention

The ORG context stores personal data (employee names, dates of birth, addresses, phone numbers, work contacts). "Don't log secrets" (§42) is necessary but not sufficient — treat PII as a first-class concern.

**Classification.** Tag fields by sensitivity:

```text
PUBLIC        codes, names of legal entities
INTERNAL      org structure, non-personal metadata
PII           employee name, DOB, personal phone/email, home address
SENSITIVE     government IDs (PAN/TAN), tax numbers, bank details
```

**Handling rules.**

```text
- Encrypt SENSITIVE fields at rest; org_tenant_setting.is_encrypted already exists — extend
  the pattern (application-level Fernet/KMS) to employee government IDs and bank details.
- Never place PII in URLs, query strings, logs, traces, cache keys, or event payloads
  (events carry identifiers + non-personal facts only — see §24).
- Mask PII in non-production environments (seed/anonymize on refresh).
- Access to PII is itself an audited action (§43).
```

**Retention & erasure.**

```text
- Define a retention period per data class (often driven by tax/labour law, e.g. 7–8 years
  for employment records) rather than deleting on demand.
- Support right-to-erasure requests: soft-delete + crypto-shred (drop the field's key) rather
  than hard-delete, so referential history and financial records stay intact.
- A terminated employee is deactivated (status=INACTIVE), not physically removed, when linked
  to posted transactions — same rule as companies (§13).
```

---

## Appendix F — Identifier & Code Generation

Business codes (`company.code`, `employee_code`, warehouse/branch codes) are natural keys — generating them safely under concurrency needs a deliberate strategy.

**Do not** derive codes from `COUNT(*) + 1` or client-supplied sequence numbers; both race under concurrent inserts and collide after soft delete.

**Preferred approaches.**

```text
- Human-entered codes: accept from the user, validate against the partial unique index (§11.1),
  and return a clean 409 on conflict rather than pre-checking then inserting (TOCTOU).
- System-generated codes: use a per-tenant sequence/counter table updated in the same
  transaction, or a PostgreSQL sequence per tenant/type where gaps are acceptable.
```

Per-tenant counter sketch:

```text
org_code_counter(tenant_id, code_type, next_value)   -- PRIMARY KEY (tenant_id, code_type)

BEGIN
  UPDATE org_code_counter
     SET next_value = next_value + 1
   WHERE tenant_id = :t AND code_type = :ct
  RETURNING next_value;         -- row lock serializes concurrent allocators
  -- format e.g. EMP-000123, then INSERT the entity in the same transaction
COMMIT
```

Rules:

```text
- Allocate the code inside the same transaction that inserts the entity.
- Treat generated codes as display identifiers, never as the primary key (UUID stays the PK, §9).
- Sequences may leave gaps on rollback; that is acceptable — never reuse a number to "fill" a gap.
- Formatting (prefix, width) is configuration, not hard-coded.
```

---

**End of JeslotERP Organization Platform Developer Integration Guide**
