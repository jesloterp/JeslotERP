# JeslotERP Organization Platform — Complete API Endpoint Specification

**Version:** 1.2  
**Last reviewed:** 2026-09-12  
**Status:** **SoR-Live** — routers are thin; UoM + FX under `/reference`. Not Production.  
**Base Path:** `/api/v1/org`  
**Format:** JSON  
**Authentication:** OAuth2/OIDC Bearer Access Token issued by JeslotERP IAM  
**Authorization:** IAM permission + ORG hierarchy validation  
**Primary database:** PostgreSQL

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial production API baseline. |
| 1.1 | 2026-09-08 | Contact API given a guarded polymorphic owner + owner-scoped lists (§27); Address API reconciled with the shared-entity + composite-FK model (§26); removed ORG-owned `/context/switch` from the inventory (IAM owns it, §34); added the missing `/companies/{id}/departments`, `/designations`, `/fiscal-years` list endpoints; standardized pagination, reference-API auth, and 429 `Retry-After` (§45, §28, §49). |
| 1.2 | 2026-09-12 | TASK-SOR-027: UoM + FX under `/reference` (stay in p02). Empty catalog is `[]`. |

---

# 1. API Design Principles

All organization APIs follow these rules:

1. The authenticated token is the authoritative source of `tenant_id`, `company_id`, and `branch_id`.
2. Client-supplied tenant/company/branch IDs are never trusted for authorization.
3. Every resource query is scoped to the authenticated organization context.
4. IAM owns authentication, users, roles and permissions.
5. ORG owns organizational resources.
6. Public APIs never expose SQLAlchemy models.
7. IDs are UUIDs.
8. Mutating endpoints support optimistic concurrency where applicable.
9. Important mutations generate domain events and transactional-outbox records.
10. Soft deletion is preferred for master data.
11. API errors use one standard error envelope.
12. Internal integration APIs are separated from customer-facing APIs.

---

# 2. Authentication

Every protected endpoint requires:

```http
Authorization: Bearer <access_token>
```

The access token contains the current security context:

```json
{
  "iss": "https://idp.jesloterp.example",
  "sub": "USER_UUID",
  "aud": "jeslot-api",
  "tenant_id": "TENANT_UUID",
  "company_id": "COMPANY_UUID",
  "branch_id": "BRANCH_UUID",
  "session_id": "SESSION_UUID",
  "scope": "openid profile org",
  "iat": 1770000000,
  "exp": 1770003600
}
```

Do not use:

```http
X-Tenant-ID
X-Company-ID
X-Branch-ID
```

as authoritative security controls.

---

# 3. Common Headers

Recommended:

```http
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
X-Request-ID: <uuid>
Idempotency-Key: <unique-key>
If-Match: <version-or-etag>
```

`Idempotency-Key` is required for selected retry-sensitive POST operations.

`If-Match` should be used on high-value updates where optimistic concurrency is enabled.

---

# 4. Standard Error Contract

All errors should follow:

```json
{
  "error": {
    "code": "ORG_COMPANY_NOT_FOUND",
    "message": "Company was not found.",
    "details": {},
    "request_id": "UUID"
  }
}
```

Common errors:

```text
ORG_TENANT_NOT_FOUND
ORG_COMPANY_NOT_FOUND
ORG_BRANCH_NOT_FOUND
ORG_EMPLOYEE_NOT_FOUND
ORG_WAREHOUSE_NOT_FOUND
ORG_PERMISSION_DENIED
ORG_CONTEXT_INVALID
ORG_CROSS_TENANT_ACCESS
ORG_CROSS_COMPANY_ACCESS
ORG_CROSS_BRANCH_ACCESS
ORG_DUPLICATE_CODE
ORG_VERSION_CONFLICT
ORG_RESOURCE_INACTIVE
ORG_RESOURCE_DELETED
ORG_INVALID_HIERARCHY
ORG_FISCAL_PERIOD_CLOSED
ORG_TAX_REGISTRATION_INVALID
ORG_IDEMPOTENCY_CONFLICT
```

---

# 5. Health and Metadata

These endpoints are generally public or restricted to infrastructure.

## GET `/health/live`

Purpose:

Confirm that the application process is alive.

Response:

```json
{
  "status": "ok"
}
```

---

## GET `/health/ready`

Purpose:

Confirm that the service can accept traffic.

Checks may include:

- PostgreSQL
- required configuration
- required infrastructure dependencies

Response:

```json
{
  "status": "ready"
}
```

Do not expose credentials or detailed infrastructure information.

---

## GET `/version`

Response:

```json
{
  "service": "organization",
  "version": "1.0.0",
  "api_version": "v1"
}
```

---

# 6. Tenant APIs

Tenant is the highest organizational scope.

## GET `/tenants`

Permission:

```text
org.tenant.read
```

Purpose:

List tenants available to the authenticated platform administrator or authorized caller.

Query parameters:

```text
search
status
tenant_type
page
page_size
cursor
sort
```

Response:

```json
{
  "items": [
    {
      "id": "UUID",
      "code": "TEN001",
      "name": "Example Tenant",
      "display_name": "Example",
      "tenant_type": "STANDARD",
      "status": "ACTIVE"
    }
  ],
  "next_cursor": null
}
```

---

## POST `/tenants`

Permission:

```text
org.tenant.create
```

Request:

```json
{
  "code": "TEN001",
  "name": "Example Tenant",
  "display_name": "Example",
  "tenant_type": "STANDARD",
  "email": "admin@example.com",
  "phone": "+919999999999",
  "country_id": "UUID",
  "default_currency_id": "UUID",
  "default_language_id": "UUID",
  "default_timezone_id": "UUID"
}
```

Behavior:

1. Validate caller.
2. Validate unique tenant code.
3. Validate reference masters.
4. Create tenant.
5. Create default tenant settings if required.
6. Write `TenantCreated` outbox event.
7. Commit transaction.

Response:

```json
{
  "id": "UUID",
  "code": "TEN001",
  "name": "Example Tenant",
  "status": "ACTIVE"
}
```

---

## GET `/tenants/{tenant_id}`

Permission:

```text
org.tenant.read
```

Returns one tenant.

Security:

```text
tenant_id must be accessible to caller
```

---

## PATCH `/tenants/{tenant_id}`

Permission:

```text
org.tenant.update
```

Request:

```json
{
  "name": "Updated Tenant",
  "display_name": "Updated"
}
```

Recommended header:

```http
If-Match: <version>
```

Response:

Updated tenant representation.

---

## POST `/tenants/{tenant_id}/activate`

Permission:

```text
org.tenant.activate
```

Activates a tenant.

---

## POST `/tenants/{tenant_id}/deactivate`

Permission:

```text
org.tenant.deactivate
```

Deactivates a tenant.

Do not physically delete a tenant containing business data.

---

## DELETE `/tenants/{tenant_id}`

Permission:

```text
org.tenant.delete
```

Recommended behavior:

Soft-delete only.

Response:

```text
204 No Content
```

---

# 7. Tenant Settings APIs

## GET `/tenants/{tenant_id}/settings`

Returns tenant settings visible to the caller.

---

## PUT `/tenants/{tenant_id}/settings/{key}`

Permission:

```text
org.tenant.settings.update
```

Request:

```json
{
  "value": {
    "some_setting": true
  }
}
```

Never use this API to store raw database passwords, JWT keys, refresh tokens or other secrets.

---

## DELETE `/tenants/{tenant_id}/settings/{key}`

Deletes a non-system tenant setting.

---

# 8. Tenant Branding APIs

## GET `/tenants/{tenant_id}/branding`

Returns:

```json
{
  "is_white_label": true,
  "brand_name": "Example ERP",
  "brand_logo_file_id": "FILE_UUID",
  "brand_favicon_file_id": "FILE_UUID",
  "brand_primary_color": "#123456",
  "brand_secondary_color": "#654321",
  "brand_domain": "erp.example.com"
}
```

---

## PUT `/tenants/{tenant_id}/branding`

Permission:

```text
org.tenant.branding.update
```

Request:

```json
{
  "is_white_label": true,
  "brand_name": "Example ERP",
  "brand_logo_file_id": "FILE_UUID",
  "brand_favicon_file_id": "FILE_UUID",
  "brand_primary_color": "#123456",
  "brand_secondary_color": "#654321",
  "brand_font": "Inter",
  "brand_domain": "erp.example.com"
}
```

---

# 9. Tenant Subscription APIs

Subscription/billing should preferably be owned by the platform billing service. ORG exposes only the organization-facing reference.

## GET `/tenants/{tenant_id}/subscription`

Returns:

```json
{
  "plan_code": "PRO",
  "status": "ACTIVE",
  "starts_at": "2026-01-01T00:00:00Z",
  "ends_at": null
}
```

---

## POST `/internal/tenants/{tenant_id}/subscription`

Internal service endpoint.

Used by billing/platform service.

Authentication:

```text
service-to-service OAuth2
```

---

# 10. Tenant Deployment APIs

These are internal/platform-admin APIs.

## GET `/internal/tenants/{tenant_id}/deployment`

Returns deployment metadata.

Do not return database credentials.

---

## PUT `/internal/tenants/{tenant_id}/deployment`

Request:

```json
{
  "deployment_mode": "SHARED",
  "storage_limit_mb": 10240,
  "database_provider": "POSTGRESQL",
  "database_host_ref": "secret://...",
  "database_name_ref": "secret://...",
  "database_schema": "tenant_schema"
}
```

The actual secret remains in a secrets manager.

---

# 11. Company APIs

Company represents a legal entity inside a tenant.

## GET `/companies`

Permission:

```text
org.company.read
```

Query:

```text
search
status
company_type
page
page_size
cursor
sort
```

Response:

```json
{
  "items": [
    {
      "id": "UUID",
      "tenant_id": "UUID",
      "code": "COMP001",
      "name": "Example Pvt Ltd",
      "legal_name": "Example Private Limited",
      "status": "ACTIVE"
    }
  ],
  "next_cursor": null
}
```

---

## POST `/companies`

Permission:

```text
org.company.create
```

Request:

```json
{
  "code": "COMP001",
  "name": "Example Pvt Ltd",
  "legal_name": "Example Private Limited",
  "company_type": "PRIVATE_LIMITED",
  "registration_number": "12345",
  "pan_number": "ABCDE1234F",
  "currency_id": "UUID",
  "language_id": "UUID",
  "timezone_id": "UUID",
  "country_id": "UUID",
  "state_id": "UUID",
  "city_id": "UUID",
  "address_id": "UUID"
}
```

`tenant_id` is taken from the authenticated context, not blindly from the request body.

---

## GET `/companies/{company_id}`

Returns a company within the current tenant.

---

## PATCH `/companies/{company_id}`

Permission:

```text
org.company.update
```

Uses optimistic concurrency.

---

## POST `/companies/{company_id}/activate`

Permission:

```text
org.company.activate
```

---

## POST `/companies/{company_id}/deactivate`

Permission:

```text
org.company.deactivate
```

---

## DELETE `/companies/{company_id}`

Permission:

```text
org.company.delete
```

Soft delete.

---

## GET `/companies/{company_id}/summary`

Returns an optimized read model:

```json
{
  "company": {},
  "branch_count": 4,
  "employee_count": 125,
  "warehouse_count": 3,
  "active_tax_registrations": 2,
  "current_fiscal_year": {}
}
```

---

# 12. Branch APIs

## GET `/branches`

Filters:

```text
company_id
search
status
branch_type
page
page_size
cursor
```

The supplied `company_id` must belong to the current tenant and caller context.

---

## POST `/branches`

Permission:

```text
org.branch.create
```

Request:

```json
{
  "company_id": "UUID",
  "code": "BR001",
  "name": "Ahmedabad Branch",
  "branch_type": "OFFICE",
  "parent_branch_id": null,
  "email": "ahmedabad@example.com",
  "phone": "+917900000000",
  "address_id": "UUID",
  "is_head_branch": true
}
```

Validation:

```text
company belongs to tenant
parent branch belongs to same company
```

---

## GET `/branches/{branch_id}`

---

## PATCH `/branches/{branch_id}`

---

## POST `/branches/{branch_id}/activate`

---

## POST `/branches/{branch_id}/deactivate`

---

## DELETE `/branches/{branch_id}`

Soft delete.

---

## GET `/companies/{company_id}/branches`

Returns all branches of a company.

---

## GET `/branches/{branch_id}/organization`

Returns:

```text
Tenant
Company
Branch
Warehouse summary
Employee summary
```

---

# 13. Department APIs

## GET `/departments`

Filters:

```text
company_id
search
status
```

---

## POST `/departments`

Request:

```json
{
  "company_id": "UUID",
  "code": "FIN",
  "name": "Finance",
  "parent_department_id": null,
  "description": "Finance department"
}
```

Validation:

```text
company belongs to tenant
parent department belongs to same company
```

---

## GET `/departments/{department_id}`

---

## PATCH `/departments/{department_id}`

---

## POST `/departments/{department_id}/activate`

---

## POST `/departments/{department_id}/deactivate`

---

## DELETE `/departments/{department_id}`

---

## GET `/companies/{company_id}/departments`

Flat, company-scoped department list (the `company_id` must belong to the caller's tenant).

---

## GET `/companies/{company_id}/departments/tree`

Returns hierarchical department structure.

Example:

```json
{
  "items": [
    {
      "id": "UUID",
      "code": "FIN",
      "name": "Finance",
      "children": [
        {
          "id": "UUID",
          "code": "AP",
          "name": "Accounts Payable",
          "children": []
        }
      ]
    }
  ]
}
```

---

# 14. Designation APIs

## GET `/designations`

---

## POST `/designations`

Request:

```json
{
  "company_id": "UUID",
  "code": "MGR",
  "name": "Manager",
  "level": 5,
  "parent_designation_id": null,
  "description": "Management role"
}
```

---

## GET `/designations/{designation_id}`

---

## PATCH `/designations/{designation_id}`

---

## POST `/designations/{designation_id}/activate`

---

## POST `/designations/{designation_id}/deactivate`

---

## DELETE `/designations/{designation_id}`

---

## GET `/companies/{company_id}/designations`

Flat, company-scoped designation list (the `company_id` must belong to the caller's tenant).

---

## GET `/companies/{company_id}/designations/tree`

Returns designation hierarchy.

---

# 15. Employee APIs

Employee is the ORG employment record. IAM remains the owner of identity.

## GET `/employees`

Filters:

```text
company_id
branch_id
department_id
designation_id
employee_type
status
search
page
page_size
cursor
```

---

## POST `/employees`

Permission:

```text
org.employee.create
```

Request:

```json
{
  "user_id": "IAM_USER_UUID",
  "employee_code": "EMP001",
  "company_id": "UUID",
  "branch_id": "UUID",
  "department_id": "UUID",
  "designation_id": "UUID",
  "joining_date": "2026-09-01",
  "employee_type": "FULL_TIME",
  "work_email": "employee@example.com",
  "work_phone": "+919999999999",
  "default_cost_center_id": "UUID",
  "default_profit_center_id": "UUID"
}
```

Validation:

```text
IAM user exists
user is accessible to tenant
company belongs to tenant
branch belongs to company
department belongs to company
designation belongs to company
cost center belongs to company
profit center belongs to company
```

---

## GET `/employees/{employee_id}`

---

## PATCH `/employees/{employee_id}`

---

## POST `/employees/{employee_id}/activate`

---

## POST `/employees/{employee_id}/deactivate`

---

## DELETE `/employees/{employee_id}`

Soft delete.

---

## PUT `/employees/{employee_id}/assignment`

Changes:

```text
company
branch
department
designation
manager
cost center
profit center
```

Must validate the entire hierarchy atomically.

> **Note:** the schema currently constrains `reporting_manager` to the **same branch** (composite FK). If cross-branch / matrix reporting is required, the schema FK must be widened first (see developer guide §36 note and schema note #10) — the API cannot assign a manager from another branch until then.

---

## GET `/employees/{employee_id}/organization`

Returns:

```json
{
  "employee": {},
  "company": {},
  "branch": {},
  "department": {},
  "designation": {},
  "manager": {}
}
```

---

## GET `/employees/{employee_id}/manager`

Returns the reporting manager.

---

## GET `/employees/{employee_id}/subordinates`

Returns employees reporting to the employee.

---

## GET `/companies/{company_id}/employees`

Company-scoped employee list.

---

## GET `/branches/{branch_id}/employees`

Branch-scoped employee list.

---

# 16. Cost Center APIs

## GET `/cost-centers`

---

## POST `/cost-centers`

Request:

```json
{
  "company_id": "UUID",
  "code": "CC001",
  "name": "Operations",
  "parent_cost_center_id": null
}
```

---

## GET `/cost-centers/{cost_center_id}`

---

## PATCH `/cost-centers/{cost_center_id}`

---

## POST `/cost-centers/{cost_center_id}/activate`

---

## POST `/cost-centers/{cost_center_id}/deactivate`

---

## DELETE `/cost-centers/{cost_center_id}`

---

## GET `/companies/{company_id}/cost-centers/tree`

Returns hierarchy.

---

# 17. Profit Center APIs

## GET `/profit-centers`

---

## POST `/profit-centers`

Request:

```json
{
  "company_id": "UUID",
  "code": "PC001",
  "name": "North Region",
  "parent_profit_center_id": null
}
```

---

## GET `/profit-centers/{profit_center_id}`

---

## PATCH `/profit-centers/{profit_center_id}`

---

## POST `/profit-centers/{profit_center_id}/activate`

---

## POST `/profit-centers/{profit_center_id}/deactivate`

---

## DELETE `/profit-centers/{profit_center_id}`

---

## GET `/companies/{company_id}/profit-centers/tree`

---

# 18. Warehouse APIs

## GET `/warehouses`

Filters:

```text
company_id
branch_id
warehouse_type
status
search
```

---

## POST `/warehouses`

Request:

```json
{
  "company_id": "UUID",
  "branch_id": "UUID",
  "code": "WH001",
  "name": "Main Warehouse",
  "warehouse_type": "GENERAL",
  "manager_employee_id": "UUID",
  "address_id": "UUID",
  "allow_negative_stock": false
}
```

Validation:

```text
company belongs to tenant
branch belongs to company
manager belongs to same company/branch
```

---

## GET `/warehouses/{warehouse_id}`

---

## PATCH `/warehouses/{warehouse_id}`

---

## POST `/warehouses/{warehouse_id}/activate`

---

## POST `/warehouses/{warehouse_id}/deactivate`

---

## DELETE `/warehouses/{warehouse_id}`

---

## GET `/branches/{branch_id}/warehouses`

---

# 19. Warehouse Location APIs

## GET `/warehouses/{warehouse_id}/locations`

---

## POST `/warehouses/{warehouse_id}/locations`

Request:

```json
{
  "code": "A-01",
  "name": "Aisle 01",
  "location_type": "BIN",
  "barcode": "890000001",
  "parent_location_id": "UUID"
}
```

---

## GET `/locations/{location_id}`

---

## PATCH `/locations/{location_id}`

---

## POST `/locations/{location_id}/activate`

---

## POST `/locations/{location_id}/deactivate`

---

## DELETE `/locations/{location_id}`

---

## GET `/warehouses/{warehouse_id}/locations/tree`

Returns:

```text
Warehouse
 ├── Zone A
 │    ├── A-01
 │    └── A-02
 └── Zone B
      └── B-01
```

---

# 20. Fiscal Year APIs

## GET `/fiscal-years`

Filters:

```text
company_id
is_current
is_closed
status
```

---

## POST `/fiscal-years`

Request:

```json
{
  "company_id": "UUID",
  "code": "FY2026-27",
  "name": "Financial Year 2026-27",
  "start_date": "2026-04-01",
  "end_date": "2027-03-31"
}
```

Validation:

```text
start_date < end_date
company belongs to tenant
period ranges must not overlap
```

---

## GET `/fiscal-years/{fiscal_year_id}`

---

## PATCH `/fiscal-years/{fiscal_year_id}`

Only permitted before financial activity begins.

---

## POST `/fiscal-years/{fiscal_year_id}/set-current`

Permission:

```text
org.fiscal_year.set_current
```

Only one current fiscal year should exist per company.

---

## POST `/fiscal-years/{fiscal_year_id}/close`

Permission:

```text
org.fiscal_year.close
```

Must validate that all required periods/processes permit closure.

---

## GET `/companies/{company_id}/fiscal-years`

Company-scoped fiscal-year list (the `company_id` must belong to the caller's tenant).

---

## GET `/companies/{company_id}/fiscal-years/current`

Returns the current fiscal year.

---

# 21. Financial Period APIs

## GET `/financial-periods`

Filters:

```text
fiscal_year_id
company_id
status
period_no
```

---

## POST `/financial-periods`

Request:

```json
{
  "fiscal_year_id": "UUID",
  "period_no": 1,
  "period_name": "April 2026",
  "start_date": "2026-04-01",
  "end_date": "2026-04-30",
  "period_type": "MONTHLY"
}
```

---

## GET `/financial-periods/{period_id}`

---

## PATCH `/financial-periods/{period_id}`

Only allowed where business rules permit.

---

## POST `/financial-periods/{period_id}/soft-close`

Permission:

```text
org.finance.period.soft_close
```

---

## POST `/financial-periods/{period_id}/close`

Permission:

```text
org.finance.period.close
```

Creates an auditable closure.

---

## POST `/financial-periods/{period_id}/reopen`

Permission:

```text
org.finance.period.reopen
```

Request:

```json
{
  "reason": "Correction required for statutory adjustment"
}
```

Must be strongly authorized and audited.

---

## GET `/financial-periods/current`

Returns the active period for the current company/context.

---

## GET `/companies/{company_id}/financial-periods/current`

Returns current company period.

---

# 22. Fiscal Period Closure APIs

## GET `/financial-periods/{period_id}/closures`

Returns closure history/status.

---

## POST `/financial-periods/{period_id}/closures`

Creates a scoped closure operation.

Request:

```json
{
  "branch_id": "UUID",
  "reason": "Month-end closing"
}
```

---

## GET `/financial-periods/{period_id}/closure-status`

Returns:

```json
{
  "status": "CLOSED",
  "branch_id": "UUID",
  "closed_at": "2026-05-01T10:00:00Z",
  "closed_by": "UUID"
}
```

---

# 23. Working Calendar APIs

## GET `/working-calendars`

---

## POST `/working-calendars`

Request:

```json
{
  "company_id": "UUID",
  "name": "India Standard Calendar",
  "week_start": 1,
  "working_days": [1, 2, 3, 4, 5, 6],
  "working_hours": {
    "start": "09:00",
    "end": "18:00"
  },
  "timezone_id": "UUID"
}
```

---

## GET `/working-calendars/{calendar_id}`

---

## PATCH `/working-calendars/{calendar_id}`

---

## DELETE `/working-calendars/{calendar_id}`

---

## GET `/companies/{company_id}/working-calendars`

---

# 24. Holiday APIs

## GET `/working-calendars/{calendar_id}/holidays`

Query:

```text
from_date
to_date
holiday_type
is_optional
```

---

## POST `/working-calendars/{calendar_id}/holidays`

Request:

```json
{
  "holiday_date": "2026-08-15",
  "holiday_name": "Independence Day",
  "holiday_type": "PUBLIC",
  "is_optional": false
}
```

---

## GET `/holidays/{holiday_id}`

---

## PATCH `/holidays/{holiday_id}`

---

## DELETE `/holidays/{holiday_id}`

---

## POST `/working-calendars/{calendar_id}/holidays/bulk`

For bulk holiday import.

Use idempotency and background processing for large imports.

---

# 25. Tax Registration APIs

This is especially important for Indian ERP/GST.

## GET `/tax-registrations`

Filters:

```text
company_id
branch_id
registration_type
status
state_id
```

---

## POST `/tax-registrations`

Request:

```json
{
  "company_id": "UUID",
  "branch_id": "UUID",
  "registration_type": "REGULAR",
  "registration_number": "24ABCDE1234F1Z5",
  "state_id": "UUID",
  "effective_from": "2026-04-01",
  "is_primary": true
}
```

Validation:

```text
company belongs to tenant
branch belongs to company
state is valid
registration number format valid
registration number not duplicated
```

---

## GET `/tax-registrations/{tax_registration_id}`

---

## PATCH `/tax-registrations/{tax_registration_id}`

---

## POST `/tax-registrations/{tax_registration_id}/activate`

---

## POST `/tax-registrations/{tax_registration_id}/deactivate`

---

## DELETE `/tax-registrations/{tax_registration_id}`

Soft delete.

---

## GET `/companies/{company_id}/tax-registrations`

---

## GET `/companies/{company_id}/tax-registrations/primary`

Returns the primary registration.

---

## GET `/branches/{branch_id}/tax-registrations`

---

# 26. Address APIs

Addresses use the shared-entity + typed composite-FK model (developer guide §36, Pattern A): a single `org_address` table, referenced by owners (company / branch / warehouse / tenant) via a tenant-safe `(address_id, tenant_id)` foreign key. Addresses are always created within the authenticated tenant context — `tenant_id` comes from the token, never the request body — so an address can never be linked across tenants.

## GET `/addresses`

Tenant-scoped. Filters:

```text
address_type
city_id
search
page
page_size
cursor
```

---

## POST `/addresses`

Creates an address owned by the caller's tenant. The returned `id` is then referenced by a company/branch/warehouse `address_id` field.

Request:

```json
{
  "address_type": "REGISTERED",
  "address_line1": "123 Main Road",
  "address_line2": "Second Floor",
  "landmark": "Near XYZ",
  "country_id": "UUID",
  "state_id": "UUID",
  "city_id": "UUID",
  "postal_code": "380001",
  "latitude": 23.0225,
  "longitude": 72.5714,
  "is_primary": true
}
```

---

## GET `/addresses/{address_id}`

---

## PATCH `/addresses/{address_id}`

---

## DELETE `/addresses/{address_id}`

---

# 27. Contact APIs

A contact is a guarded polymorphic satellite (developer guide §36.1): it always belongs to an owner, identified by `owner_type` + `owner_id`. `owner_type` is a closed set (`COMPANY`, `BRANCH`, `WAREHOUSE`, `TENANT`); there is no cross-table FK, so the service layer validates that `owner_id` exists within the caller's tenant. Every query is tenant-scoped and owner-scoped.

## GET `/contacts`

Filters (`owner_type` + `owner_id` are required together to scope a listing):

```text
owner_type
owner_id
contact_type
search
is_primary
page
page_size
cursor
```

---

## POST `/contacts`

Request:

```json
{
  "owner_type": "COMPANY",
  "owner_id": "OWNER_UUID",
  "contact_type": "SALES",
  "person_name": "John Doe",
  "designation": "Manager",
  "email": "john@example.com",
  "phone": "+919999999999",
  "mobile": "+919999999999",
  "is_primary": true
}
```

Validation:

```text
owner_type is a permitted value
owner_id exists and belongs to the caller's tenant
at most one primary contact per (owner_type, owner_id) among active rows
```

---

## GET `/contacts/{contact_id}`

---

## PATCH `/contacts/{contact_id}`

---

## DELETE `/contacts/{contact_id}`

Soft delete.

---

## GET `/companies/{company_id}/contacts`

Contacts owned by a company (`owner_type=COMPANY`).

---

## GET `/branches/{branch_id}/contacts`

Contacts owned by a branch (`owner_type=BRANCH`).

---

# 28. Global Country APIs

These are global reference-data APIs (§28–33). They are **read-only** and require a valid access token like every other endpoint, but they are **not tenant-scoped** — the data is global master data shared across all tenants. They are safe to cache aggressively (§ developer guide §27) since they change rarely.

## GET `/reference/countries`

Query:

```text
search
active
page
page_size
```

---

## GET `/reference/countries/{country_id}`

---

# 29. Global State APIs

## GET `/reference/states`

Filters:

```text
country_id
search
```

---

## GET `/reference/states/{state_id}`

---

# 30. Global City APIs

## GET `/reference/cities`

Filters:

```text
country_id
state_id
search
postal_code
```

---

## GET `/reference/cities/{city_id}`

---

# 31. Currency APIs

## GET `/reference/currencies`

---

## GET `/reference/currencies/{currency_id}`

---

# 32. Language APIs

## GET `/reference/languages`

---

## GET `/reference/languages/{language_id}`

---

# 33. Timezone APIs

## GET `/reference/timezones`

---

## GET `/reference/timezones/{timezone_id}`

---

# 33A. Units of measure + FX (p02 only — no p34)

Empty list on `AsyncSession` is `[]`. Convert never invents a missing factor/rate.

| Method | Path | Permission |
|---|---|---|
| GET | `/reference/uoms` | `org.uom.read` |
| POST | `/reference/uoms` | `org.uom.manage` |
| GET | `/reference/uom-conversions` | `org.uom.read` |
| POST | `/reference/uom-conversions` | `org.uom.manage` |
| POST | `/reference/uoms/convert` | `org.uom.read` |
| GET | `/reference/fx-rates` | `org.fx.read` |
| POST | `/reference/fx-rates` | `org.fx.manage` |
| POST | `/reference/fx/convert` | `org.fx.read` |

`POST /uoms/convert` body: `{amount, from_uom, to_uom}`. Missing pair → 400.  
`POST /fx/convert` body: `{amount, from_currency, to_currency, rate_type?, as_of?}`. Missing rate → 400.

---

# 34. Current Organization Context APIs

These APIs are useful for ERP UI.

## GET `/context/current`

Returns the context embedded in the current access token plus resolved names.

Example:

```json
{
  "tenant": {
    "id": "UUID",
    "name": "Example Tenant"
  },
  "company": {
    "id": "UUID",
    "name": "Example Pvt Ltd"
  },
  "branch": {
    "id": "UUID",
    "name": "Ahmedabad Branch"
  }
}
```

---

## GET `/context/organization-tree`

Returns the organizations available to the current user.

Example:

```json
{
  "tenants": [
    {
      "id": "TENANT_A",
      "name": "Tenant A",
      "companies": [
        {
          "id": "COMPANY_A1",
          "name": "Company A1",
          "branches": [
            {
              "id": "BRANCH_A1",
              "name": "Branch A1"
            }
          ]
        }
      ]
    }
  ]
}
```

The availability of tenants/companies/branches is determined through IAM assignments.

---

## POST `/context/validate`

Request:

```json
{
  "tenant_id": "UUID",
  "company_id": "UUID",
  "branch_id": "UUID"
}
```

This endpoint validates hierarchy only.

It does not authorize a user by itself.

---

## Context switch (IAM-owned — not an ORG endpoint)

`POST /context/switch` is **owned by IAM**, because switching context creates a new access token. ORG does not expose this endpoint and must never mint IAM tokens.

ORG only exposes an internal validation endpoint that IAM calls during a switch:

```text
POST /internal/context/validate
```

IAM performs:

```text
assignment validation
+
ORG hierarchy validation
+
permission resolution
+
new token issuance
```

ORG should not independently mint IAM access tokens.

---

# 35. Organization Tree APIs

## GET `/organization/tree`

Returns:

```text
Tenant
 ├── Company
 │    ├── Branch
 │    │    ├── Warehouse
 │    │    └── Employees
 │    ├── Department
 │    ├── Designation
 │    ├── Cost Center
 │    ├── Profit Center
 │    └── Fiscal Year
```

Query:

```text
include=branches,warehouses,employees,departments
```

Do not recursively load huge ORM graphs. Use an optimized query/read model.

---

# 36. Search APIs

## GET `/search`

Permission:

```text
org.search
```

Query:

```text
q
entity
limit
```

Example:

```text
GET /api/v1/org/search?q=ABC&entity=company
```

Supported entities:

```text
tenant
company
branch
employee
department
designation
warehouse
cost_center
profit_center
```

The search must always respect the authenticated context.

---

# 37. Bulk APIs

Bulk operations should be explicit.

## POST `/companies/bulk`

---

## POST `/branches/bulk`

---

## POST `/employees/bulk`

---

## POST `/departments/bulk`

---

## POST `/warehouses/bulk`

For large files, return:

```json
{
  "job_id": "UUID",
  "status": "QUEUED"
}
```

Then:

```text
GET /jobs/{job_id}
```

should expose job status through the platform job service rather than making the ORG request synchronous.

---

# 38. Import Validation

## POST `/imports/validate`

Request:

```json
{
  "entity": "employee",
  "file_id": "FILE_UUID"
}
```

Response:

```json
{
  "job_id": "UUID",
  "status": "QUEUED"
}
```

---

# 39. Internal Integration APIs

These endpoints are for trusted services, not browser clients.

Prefix:

```text
/api/v1/org/internal
```

Authentication:

```text
service OAuth2/client credentials
```

Examples:

```text
GET  /internal/tenants/{tenant_id}
GET  /internal/companies/{company_id}
GET  /internal/branches/{branch_id}
GET  /internal/employees/{employee_id}
GET  /internal/warehouses/{warehouse_id}

POST /internal/context/validate
POST /internal/hierarchy/validate
```

---

# 40. Hierarchy Validation API

## POST `/internal/hierarchy/validate`

Request:

```json
{
  "tenant_id": "UUID",
  "company_id": "UUID",
  "branch_id": "UUID",
  "department_id": "UUID",
  "warehouse_id": "UUID"
}
```

Response:

```json
{
  "valid": true
}
```

If invalid:

```json
{
  "valid": false,
  "error": {
    "code": "ORG_INVALID_HIERARCHY",
    "message": "Warehouse does not belong to the specified branch."
  }
}
```

This is useful for Finance, Sales and Inventory.

---

# 41. Reference Resolution APIs

## POST `/internal/resolve`

Request:

```json
{
  "entity": "company",
  "ids": [
    "UUID1",
    "UUID2",
    "UUID3"
  ]
}
```

Response:

```json
{
  "items": [
    {
      "id": "UUID1",
      "code": "COMP001",
      "name": "Example Pvt Ltd",
      "status": "ACTIVE"
    }
  ]
}
```

Useful for microservice read models.

---

# 42. Organization Status APIs

## GET `/companies/{company_id}/status`

---

## GET `/branches/{branch_id}/status`

---

## GET `/employees/{employee_id}/status`

---

## GET `/warehouses/{warehouse_id}/status`

These can be optimized lightweight endpoints for other services.

---

# 43. Audit APIs

Audit event creation should normally happen internally, not through arbitrary public CRUD.

## GET `/audit-events`

Permissions:

```text
org.audit.read
```

Filters:

```text
from
to
user_id
action
resource_type
resource_id
severity
```

---

## GET `/audit-events/{event_id}`

Never provide a public:

```text
POST /audit-events
```

for arbitrary audit creation.

Application code creates audit records through the audit service.

---

# 44. Organization Events

Events emitted by ORG include:

```text
org.tenant.created
org.tenant.updated
org.tenant.deactivated

org.company.created
org.company.updated
org.company.deactivated

org.branch.created
org.branch.updated
org.branch.deactivated

org.employee.created
org.employee.updated
org.employee.deactivated

org.department.created
org.department.updated

org.designation.created
org.designation.updated

org.warehouse.created
org.warehouse.updated
org.warehouse.deactivated

org.fiscal_year.created
org.fiscal_year.closed

org.financial_period.created
org.financial_period.closed
org.financial_period.reopened

org.tax_registration.created
org.tax_registration.updated
org.tax_registration.deactivated
```

Events are not HTTP APIs. They are published through the transactional outbox.

---

# 45. Pagination Contract

Offset pagination:

```text
?page=1&page_size=50
```

Response:

```json
{
  "items": [],
  "page": 1,
  "page_size": 50,
  "total": 1000
}
```

For large tables prefer cursor pagination:

```text
?cursor=abc&limit=50
```

Response:

```json
{
  "items": [],
  "next_cursor": "xyz"
}
```

Maximum page size:

```text
100
```

unless a specific API has a justified different limit.

**Which contract to use per endpoint (pick one, don't mix on the same endpoint):**

```text
- Small, bounded reference/config lists (currencies, timezones, a company's departments):
  offset pagination with `total`.
- Large, tenant-scoped collections (employees, audit events, contacts): cursor/keyset
  pagination with `next_cursor` — offset degrades and can skip/duplicate rows under writes.
```

Each endpoint's documentation must state which contract it returns so clients don't assume `total` is always present.

---

# 46. Sorting

Supported example:

```text
?sort=name
?sort=-created_at
```

Only allow an explicit whitelist of sortable fields.

Never directly concatenate arbitrary user-provided SQL identifiers.

---

# 47. Optimistic Concurrency

Optimistic concurrency uses the integer `version` column (developer guide §14 / schema `version_id_col`). By convention this platform carries that version in `If-Match` as a bare integer (not a quoted HTTP ETag); document it so clients send the right value.

For resources with versioning:

```http
PATCH /companies/{company_id}
If-Match: 12
```

If the record is already version 13:

```text
409 Conflict
```

Response:

```json
{
  "error": {
    "code": "ORG_VERSION_CONFLICT",
    "message": "The company was modified by another user.",
    "details": {
      "expected_version": 12,
      "current_version": 13
    },
    "request_id": "UUID"
  }
}
```

---

# 48. Idempotency

Use:

```http
Idempotency-Key: 4f2a...
```

for:

```text
POST /tenants
POST /companies
POST /branches
POST /employees
POST /warehouses
POST /tax-registrations
bulk operations
```

If the same key is replayed with the same request:

```text
return original result
```

If the same key is reused with a different request:

```text
409 ORG_IDEMPOTENCY_CONFLICT
```

---

# 49. HTTP Status Rules

```text
200 OK
201 Created
202 Accepted
204 No Content

400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Entity
429 Too Many Requests
500 Internal Server Error
```

Use `202` for asynchronous imports/jobs.

A `429 Too Many Requests` response must include a `Retry-After` header (seconds) telling the client when to retry:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

---

# 50. Security Rules for Every Endpoint

Every endpoint must answer:

```text
Who is the caller?
What tenant are they in?
What company are they in?
What branch are they in?
What permission do they have?
Does the target resource belong to that context?
Is the resource active?
Is the operation allowed in the current business state?
```

Example:

```python
await authorization.authorize(
    context,
    "org.employee.update",
)

await organization_gateway.validate_hierarchy(
    tenant_id=context.tenant_id,
    company_id=context.company_id,
    branch_id=context.branch_id,
)
```

---

# 51. Endpoint Implementation Pattern

Recommended:

```text
Router
  ↓
Pydantic Request DTO
  ↓
Command / Query
  ↓
Handler
  ↓
Domain Service
  ↓
Repository Port
  ↓
SQLAlchemy Adapter
  ↓
PostgreSQL
```

For commands:

```text
DB change
+
outbox event
+
audit event
```

should happen in one transaction where applicable.

---

# 52. Recommended Router Structure

```text
presentation/api/
├── routers/
│   ├── tenant.py
│   ├── tenant_settings.py
│   ├── tenant_branding.py
│   ├── company.py
│   ├── branch.py
│   ├── department.py
│   ├── designation.py
│   ├── employee.py
│   ├── warehouse.py
│   ├── location.py
│   ├── cost_center.py
│   ├── profit_center.py
│   ├── fiscal_year.py
│   ├── financial_period.py
│   ├── calendar.py
│   ├── holiday.py
│   ├── tax_registration.py
│   ├── address.py
│   ├── contact.py
│   ├── context.py
│   ├── search.py
│   └── internal.py
```

---

# 53. Recommended API Permission Naming

```text
org.tenant.read
org.tenant.create
org.tenant.update
org.tenant.activate
org.tenant.deactivate
org.tenant.delete

org.company.read
org.company.create
org.company.update
org.company.activate
org.company.deactivate
org.company.delete

org.branch.read
org.branch.create
org.branch.update
org.branch.activate
org.branch.deactivate
org.branch.delete

org.department.read
org.department.create
org.department.update
org.department.delete

org.designation.read
org.designation.create
org.designation.update
org.designation.delete

org.employee.read
org.employee.create
org.employee.update
org.employee.activate
org.employee.deactivate
org.employee.delete

org.warehouse.read
org.warehouse.create
org.warehouse.update
org.warehouse.activate
org.warehouse.deactivate
org.warehouse.delete

org.fiscal_year.read
org.fiscal_year.create
org.fiscal_year.update
org.fiscal_year.close
org.fiscal_year.set_current

org.finance.period.read
org.finance.period.create
org.finance.period.update
org.finance.period.soft_close
org.finance.period.close
org.finance.period.reopen

org.tax_registration.read
org.tax_registration.manage

org.address.read
org.address.manage

org.contact.read
org.contact.manage

org.audit.read
org.search
```

---

# 54. End-to-End Example — Create Company

```text
POST /api/v1/org/companies
        │
        ▼
JWT validation
        │
        ▼
RequestContext
        │
        ▼
authorize(org.company.create)
        │
        ▼
validate tenant
        │
        ▼
validate reference masters
        │
        ▼
check tenant-scoped code
        │
        ▼
CreateCompanyCommand
        │
        ▼
Company aggregate
        │
        ▼
Repository
        │
        ├── org_company INSERT
        ├── audit event
        └── outbox event
        │
        ▼
COMMIT
        │
        ▼
201 Created
```

---

# 55. End-to-End Example — Create Branch

Request:

```http
POST /api/v1/org/branches
Authorization: Bearer <token>
```

```json
{
  "company_id": "COMPANY_UUID",
  "code": "BR001",
  "name": "Ahmedabad"
}
```

Server must verify:

```text
token.tenant_id
      ↓
company belongs to token tenant
      ↓
company is active
      ↓
branch code is unique within tenant/company
      ↓
parent branch belongs to same company
      ↓
authorization
      ↓
create branch
```

The client does not choose the tenant scope independently.

---

# 56. End-to-End Example — Cross-Tenant Attack

Token:

```text
tenant_id = TENANT_A
```

Request:

```text
POST /companies
company_id = company from TENANT_B
```

Expected:

```text
403 Forbidden
```

or an equivalent non-disclosing `404`.

Never create or modify the resource.

---

# 57. End-to-End Example — Employee Assignment

Request:

```http
PUT /employees/{employee_id}/assignment
```

```json
{
  "company_id": "COMPANY_A",
  "branch_id": "BRANCH_A",
  "department_id": "DEPT_A",
  "designation_id": "DESIGNATION_A"
}
```

Server validates all relationships in one transaction.

If:

```text
DEPT_A belongs to COMPANY_B
```

return:

```text
409 or 422
ORG_INVALID_HIERARCHY
```

and make no partial update.

---

# 58. End-to-End Example — Fiscal Period Reopen

```text
POST /financial-periods/{period_id}/reopen
```

Requirements:

```text
authenticated user
+
org.finance.period.reopen
+
current tenant
+
correct company
+
period exists
+
period is CLOSED
+
reason provided
+
audit event
```

Transaction:

```text
period status update
+
closure history
+
audit
+
outbox event
```

---

# 59. APIs That Should NOT Be Public CRUD

Do not expose arbitrary CRUD endpoints for:

```text
outbox
event delivery state
internal audit writes
security event writes
database deployment credentials
database connection strings
internal authorization state
```

They should be managed by application services or internal infrastructure.

---

# 60. API Contract Testing

Every endpoint must have tests for:

### Authentication

```text
missing token
expired token
wrong issuer
wrong audience
invalid signature
```

### Authorization

```text
missing permission
wrong tenant
wrong company
wrong branch
```

### Validation

```text
invalid UUID
invalid dates
duplicate code
invalid hierarchy
inactive parent
```

### Concurrency

```text
stale version
simultaneous update
```

### Security

```text
IDOR
cross-tenant access
cross-company access
cross-branch access
privilege escalation
```

---

# 61. API Documentation

FastAPI OpenAPI should expose:

```text
/api/v1/org/openapi.json
/api/v1/org/docs
```

in development.

For production, Swagger UI may be restricted according to security policy.

Every endpoint should document:

```text
summary
description
permission
request schema
response schema
error responses
authentication
examples
```

---

# 62. Final Endpoint Inventory

```text
HEALTH
GET    /health/live
GET    /health/ready
GET    /version

TENANT
GET    /tenants
POST   /tenants
GET    /tenants/{tenant_id}
PATCH  /tenants/{tenant_id}
POST   /tenants/{tenant_id}/activate
POST   /tenants/{tenant_id}/deactivate
DELETE /tenants/{tenant_id}

TENANT SETTINGS
GET    /tenants/{tenant_id}/settings
PUT    /tenants/{tenant_id}/settings/{key}
DELETE /tenants/{tenant_id}/settings/{key}

TENANT BRANDING
GET    /tenants/{tenant_id}/branding
PUT    /tenants/{tenant_id}/branding

SUBSCRIPTION
GET    /tenants/{tenant_id}/subscription
POST   /internal/tenants/{tenant_id}/subscription

DEPLOYMENT
GET    /internal/tenants/{tenant_id}/deployment
PUT    /internal/tenants/{tenant_id}/deployment

COMPANY
GET    /companies
POST   /companies
GET    /companies/{company_id}
PATCH  /companies/{company_id}
POST   /companies/{company_id}/activate
POST   /companies/{company_id}/deactivate
DELETE /companies/{company_id}
GET    /companies/{company_id}/summary
GET    /companies/{company_id}/branches
GET    /companies/{company_id}/departments
GET    /companies/{company_id}/designations
GET    /companies/{company_id}/cost-centers/tree
GET    /companies/{company_id}/profit-centers/tree
GET    /companies/{company_id}/fiscal-years
GET    /companies/{company_id}/financial-periods/current
GET    /companies/{company_id}/tax-registrations
GET    /companies/{company_id}/tax-registrations/primary
GET    /companies/{company_id}/employees
GET    /companies/{company_id}/working-calendars

BRANCH
GET    /branches
POST   /branches
GET    /branches/{branch_id}
PATCH  /branches/{branch_id}
POST   /branches/{branch_id}/activate
POST   /branches/{branch_id}/deactivate
DELETE /branches/{branch_id}
GET    /branches/{branch_id}/organization
GET    /branches/{branch_id}/employees
GET    /branches/{branch_id}/warehouses
GET    /branches/{branch_id}/tax-registrations

DEPARTMENT
GET    /departments
POST   /departments
GET    /departments/{department_id}
PATCH  /departments/{department_id}
POST   /departments/{department_id}/activate
POST   /departments/{department_id}/deactivate
DELETE /departments/{department_id}
GET    /companies/{company_id}/departments/tree

DESIGNATION
GET    /designations
POST   /designations
GET    /designations/{designation_id}
PATCH  /designations/{designation_id}
POST   /designations/{designation_id}/activate
POST   /designations/{designation_id}/deactivate
DELETE /designations/{designation_id}
GET    /companies/{company_id}/designations/tree

EMPLOYEE
GET    /employees
POST   /employees
GET    /employees/{employee_id}
PATCH  /employees/{employee_id}
POST   /employees/{employee_id}/activate
POST   /employees/{employee_id}/deactivate
DELETE /employees/{employee_id}
PUT    /employees/{employee_id}/assignment
GET    /employees/{employee_id}/organization
GET    /employees/{employee_id}/manager
GET    /employees/{employee_id}/subordinates

COST CENTER
GET    /cost-centers
POST   /cost-centers
GET    /cost-centers/{cost_center_id}
PATCH  /cost-centers/{cost_center_id}
POST   /cost-centers/{cost_center_id}/activate
POST   /cost-centers/{cost_center_id}/deactivate
DELETE /cost-centers/{cost_center_id}
GET    /companies/{company_id}/cost-centers/tree

PROFIT CENTER
GET    /profit-centers
POST   /profit-centers
GET    /profit-centers/{profit_center_id}
PATCH  /profit-centers/{profit_center_id}
POST   /profit-centers/{profit_center_id}/activate
POST   /profit-centers/{profit_center_id}/deactivate
DELETE /profit-centers/{profit_center_id}
GET    /companies/{company_id}/profit-centers/tree

WAREHOUSE
GET    /warehouses
POST   /warehouses
GET    /warehouses/{warehouse_id}
PATCH  /warehouses/{warehouse_id}
POST   /warehouses/{warehouse_id}/activate
POST   /warehouses/{warehouse_id}/deactivate
DELETE /warehouses/{warehouse_id}
GET    /branches/{branch_id}/warehouses

LOCATION
GET    /warehouses/{warehouse_id}/locations
POST   /warehouses/{warehouse_id}/locations
GET    /locations/{location_id}
PATCH  /locations/{location_id}
POST   /locations/{location_id}/activate
POST   /locations/{location_id}/deactivate
DELETE /locations/{location_id}
GET    /warehouses/{warehouse_id}/locations/tree

FISCAL YEAR
GET    /fiscal-years
POST   /fiscal-years
GET    /fiscal-years/{fiscal_year_id}
PATCH  /fiscal-years/{fiscal_year_id}
POST   /fiscal-years/{fiscal_year_id}/set-current
POST   /fiscal-years/{fiscal_year_id}/close
GET    /companies/{company_id}/fiscal-years/current

FINANCIAL PERIOD
GET    /financial-periods
POST   /financial-periods
GET    /financial-periods/{period_id}
PATCH  /financial-periods/{period_id}
POST   /financial-periods/{period_id}/soft-close
POST   /financial-periods/{period_id}/close
POST   /financial-periods/{period_id}/reopen
GET    /financial-periods/current
GET    /companies/{company_id}/financial-periods/current

PERIOD CLOSURE
GET    /financial-periods/{period_id}/closures
POST   /financial-periods/{period_id}/closures
GET    /financial-periods/{period_id}/closure-status

WORKING CALENDAR
GET    /working-calendars
POST   /working-calendars
GET    /working-calendars/{calendar_id}
PATCH  /working-calendars/{calendar_id}
DELETE /working-calendars/{calendar_id}
GET    /companies/{company_id}/working-calendars

HOLIDAY
GET    /working-calendars/{calendar_id}/holidays
POST   /working-calendars/{calendar_id}/holidays
GET    /holidays/{holiday_id}
PATCH  /holidays/{holiday_id}
DELETE /holidays/{holiday_id}
POST   /working-calendars/{calendar_id}/holidays/bulk

TAX REGISTRATION
GET    /tax-registrations
POST   /tax-registrations
GET    /tax-registrations/{tax_registration_id}
PATCH  /tax-registrations/{tax_registration_id}
POST   /tax-registrations/{tax_registration_id}/activate
POST   /tax-registrations/{tax_registration_id}/deactivate
DELETE /tax-registrations/{tax_registration_id}
GET    /companies/{company_id}/tax-registrations
GET    /companies/{company_id}/tax-registrations/primary
GET    /branches/{branch_id}/tax-registrations

ADDRESS
GET    /addresses
POST   /addresses
GET    /addresses/{address_id}
PATCH  /addresses/{address_id}
DELETE /addresses/{address_id}

CONTACT
GET    /contacts
POST   /contacts
GET    /contacts/{contact_id}
PATCH  /contacts/{contact_id}
DELETE /contacts/{contact_id}
GET    /companies/{company_id}/contacts
GET    /branches/{branch_id}/contacts

REFERENCE
GET    /reference/countries
GET    /reference/countries/{country_id}
GET    /reference/states
GET    /reference/states/{state_id}
GET    /reference/cities
GET    /reference/cities/{city_id}
GET    /reference/currencies
GET    /reference/currencies/{currency_id}
GET    /reference/languages
GET    /reference/languages/{language_id}
GET    /reference/timezones
GET    /reference/timezones/{timezone_id}
GET    /reference/uoms
POST   /reference/uoms
GET    /reference/uom-conversions
POST   /reference/uom-conversions
POST   /reference/uoms/convert
GET    /reference/fx-rates
POST   /reference/fx-rates
POST   /reference/fx/convert

CONTEXT
GET    /context/current
GET    /context/organization-tree
POST   /context/validate
GET    /organization/tree
# NOTE: context switch is owned by IAM (issues a new token) — ORG only validates
# via POST /internal/context/validate. See §34.

SEARCH
GET    /search

BULK / IMPORT
POST   /companies/bulk
POST   /branches/bulk
POST   /employees/bulk
POST   /departments/bulk
POST   /warehouses/bulk
POST   /imports/validate

INTERNAL
GET    /internal/tenants/{tenant_id}
GET    /internal/companies/{company_id}
GET    /internal/branches/{branch_id}
GET    /internal/employees/{employee_id}
GET    /internal/warehouses/{warehouse_id}
POST   /internal/context/validate
POST   /internal/hierarchy/validate
POST   /internal/resolve

AUDIT
GET    /audit-events
GET    /audit-events/{event_id}
```

---

# 63. Recommended Implementation Priority

Implement in this order:

## Phase 1 — Foundation

```text
health
reference data
tenant
company
branch
context
```

## Phase 2 — People and Structure

```text
department
designation
employee
cost center
profit center
```

## Phase 3 — Operations

```text
warehouse
location
address
contact
```

## Phase 4 — Financial Organization

```text
fiscal year
financial period
period closure
working calendar
holiday
tax registration
```

## Phase 5 — Platform Integration

```text
internal APIs
search
bulk/import
audit
organization tree
outbox events
```

---

# 64. Production Rule

The API layer must never become the place where all business rules live.

The correct flow is:

```text
API
 ↓
Application
 ↓
Domain
 ↓
Repository
 ↓
Database
```

The API validates input shape.

The application layer coordinates the use case.

The domain enforces business rules.

The database enforces integrity.

IAM enforces identity/permissions.

ORG enforces organizational hierarchy.

This separation is the required architecture for the JeslotERP Organization Platform.

---

**End of API Specification**
