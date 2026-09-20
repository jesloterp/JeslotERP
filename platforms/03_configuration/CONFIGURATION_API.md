# JeslotERP Configuration Platform — Complete API Specification

**Version:** 1.3  
**Last reviewed:** 2026-09-08  
**Status:** **SoR-Live** — public secret GET is metadata only. Not Production.

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial API baseline. |
| 1.1 | 2026-09-08 | Standardized optimistic concurrency on the integer `version` via `If-Match` (§3, §38), matching the schema and the ORG platform; renamed the module semantic field `version → module_version` (§6) to avoid clashing with the optimistic-lock counter. |
| 1.2 | 2026-09-08 | Declared this document as HTTP source of truth; clarified UI field mapping (`ui_metadata` stored, `ui` in responses); documented service-auth expectation; marked canonical scopes path `/scopes`; aligned router layout with repo `infrastructure/http`. |
| 1.3 | 2026-09-08 | Froze package `platforms.p03_configuration` (platform number **03**). |

---

# 0. Repository Integration Contract

| Item | Frozen value |
|---|---|
| Docs folder | `docs/platforms/03_configuration/` (catalog chapter **03**) |
| Platform package | `platforms.p03_configuration` (platform number **03**) |
| PostgreSQL schema | `configuration` |
| Public base | `/api/v1/configuration` |
| Internal base | `/internal/v1/configuration` |
| Permission catalog | §5 of **this** document (canonical) |
| Scopes endpoint | `GET /api/v1/configuration/scopes` only (no `/scope`) |
| Secret plaintext resolve | `POST /internal/v1/configuration/secrets/{key}/resolve` only |
| Response UI field | API JSON field `ui` is projected from column `ui_metadata` |

Internal service authentication: IAM-issued service JWT (issuer/audience/signature + service identity + permissions). mTLS may be added at the gateway; the Configuration service itself validates the service token claims.

This API document is authoritative for HTTP paths and verbs. Schema DDL is authoritative for persistence. Developer Guide is authoritative for hexagonal layering and DoD.

## 1. API Overview

Base public API:

```text
/api/v1/configuration
```

Base internal API:

```text
/internal/v1/configuration
```

The API is metadata-driven and supports:

- Configuration modules
- Categories
- Definitions
- ENUM options
- Dynamic reference resources
- Configuration values
- Scope inheritance
- Effective configuration
- Dependencies
- Policies
- Secrets
- Templates
- Change history
- Audit
- Module/service registration
- Bulk resolution
- Cache invalidation
- Idempotent registration
- Optimistic concurrency

---

# 2. Authentication

Public APIs require:

```http
Authorization: Bearer <access_token>
```

The active organization context is derived from the authenticated token:

```json
{
  "sub": "USER_UUID",
  "tenant_id": "TENANT_UUID",
  "company_id": "COMPANY_UUID",
  "branch_id": "BRANCH_UUID",
  "session_id": "SESSION_UUID"
}
```

Do not use these as authoritative security headers:

```text
X-Tenant-ID
X-Company-ID
X-Branch-ID
```

Internal APIs require service-to-service authentication.

---

# 3. Standard Headers

Recommended:

```http
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
X-Request-ID: <uuid>
Idempotency-Key: <uuid>
If-Match: <version>
```

`Idempotency-Key` is required for retryable mutation endpoints where duplicate execution would be harmful.

`If-Match` (or the `expected_version` body field) carries the integer optimistic-lock `version` from the schema (§ FINAL_SCHEMA) — the same counter used across every JeslotERP platform. It is required for concurrency-sensitive updates. (The `row_version` UUID is an internal change token, not the `If-Match` value.)

---

# 4. Standard Response Envelope

Success:

```json
{
  "success": true,
  "data": {},
  "meta": {
    "request_id": "REQUEST_UUID"
  },
  "error": null
}
```

List:

```json
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "page_size": 25,
    "total": 100,
    "request_id": "REQUEST_UUID"
  },
  "error": null
}
```

Error:

```json
{
  "success": false,
  "data": null,
  "meta": {
    "request_id": "REQUEST_UUID"
  },
  "error": {
    "code": "CONFIGURATION_NOT_FOUND",
    "message": "Configuration definition was not found.",
    "details": {}
  }
}
```

---

# 5. Permission Model

Recommended permissions:

```text
configuration.read
configuration.write

configuration.module.read
configuration.module.register
configuration.module.write

configuration.category.read
configuration.category.write

configuration.definition.read
configuration.definition.write

configuration.option.read
configuration.option.write

configuration.resource.read
configuration.resource.register
configuration.resource.write

configuration.secret.read
configuration.secret.rotate

configuration.template.read
configuration.template.write
configuration.template.apply

configuration.audit.read
configuration.history.read
```

---

# 6. MODULE APIs

## 6.1 List Modules

```http
GET /api/v1/configuration/modules
```

Query:

```text
page
page_size
search
status
is_enabled
sort
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "id": "UUID",
      "code": "finance",
      "name": "Finance",
      "module_version": "1.0.0",
      "service_name": "finance-service",
      "status": "ACTIVE",
      "is_enabled": true
    }
  ]
}
```

---

## 6.2 Get Module

```http
GET /api/v1/configuration/modules/{module_code}
```

Example:

```http
GET /api/v1/configuration/modules/finance
```

---

## 6.3 Register Module — Internal

```http
POST /internal/v1/configuration/modules/register
```

Request:

```json
{
  "code": "finance",
  "name": "Finance",
  "description": "JeslotERP Finance module",
  "module_version": "1.0.0",
  "service_name": "finance-service",
  "service_version": "1.0.0"
}
```

Registration must be idempotent.

---

## 6.4 Update Module — Internal

```http
PUT /internal/v1/configuration/modules/{module_code}
```

---

# 7. CATEGORY APIs

## 7.1 List Categories

```http
GET /api/v1/configuration/categories
```

Filters:

```text
module
parent
status
search
```

---

## 7.2 Get Category

```http
GET /api/v1/configuration/categories/{category_code}
```

---

## 7.3 Register Category — Internal

```http
POST /internal/v1/configuration/categories/register
```

Request:

```json
{
  "module": "finance",
  "code": "tax",
  "name": "Tax",
  "description": "Finance tax settings",
  "display_order": 20
}
```

---

## 7.4 Update Category

```http
PUT /internal/v1/configuration/categories/{category_code}
```

---

# 8. CONFIGURATION DEFINITION APIs

A definition describes what a setting means.

## 8.1 List Definitions

```http
GET /api/v1/configuration/definitions
```

Query:

```text
module
category
data_type
status
is_editable
search
page
page_size
```

---

## 8.2 Get Definition

```http
GET /api/v1/configuration/definitions/{key}
```

Example:

```http
GET /api/v1/configuration/definitions/finance.tax.rounding_method
```

Response:

```json
{
  "success": true,
  "data": {
    "key": "finance.tax.rounding_method",
    "module": "finance",
    "category": "tax",
    "display_name": "Tax Rounding Method",
    "description": "Determines how tax values are rounded.",
    "data_type": "ENUM",
    "default_value": "HALF_UP",
    "allowed_scopes": [
      "TENANT",
      "COMPANY",
      "BRANCH"
    ],
    "is_required": false,
    "is_editable": true,
    "is_secret": false,
    "ui": {
      "component": "SELECT",
      "group": "Tax",
      "order": 10
    }
  }
}
```

---

## 8.3 Register Definition — Internal

```http
POST /internal/v1/configuration/definitions/register
```

Request:

```json
{
  "module": "finance",
  "category": "tax",
  "key": "finance.tax.rounding_method",
  "display_name": "Tax Rounding Method",
  "description": "Determines how tax values are rounded.",
  "data_type": "ENUM",
  "default_value": "HALF_UP",
  "allowed_scopes": [
    "TENANT",
    "COMPANY",
    "BRANCH"
  ],
  "validation_rules": {},
  "ui_metadata": {
    "component": "SELECT",
    "group": "Tax",
    "order": 10
  }
}
```

---

## 8.4 Update Definition — Internal

```http
PUT /internal/v1/configuration/definitions/{key}
```

Definition updates must be versioned.

Breaking changes must not silently alter the meaning of existing values.

---

## 8.5 Delete Definition

Prefer a lifecycle operation instead of physical deletion:

```http
DELETE /internal/v1/configuration/definitions/{key}
```

The implementation should normally soft-delete/deactivate the definition.

---

# 9. ENUM OPTION APIs

## 9.1 List Options

```http
GET /api/v1/configuration/definitions/{key}/options
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "value": "HALF_UP",
      "display_name": "Half Up",
      "labels": {
        "en": "Half Up",
        "gu": "હાફ અપ",
        "hi": "हाफ अप"
      },
      "is_active": true,
      "is_default": true,
      "display_order": 1
    },
    {
      "value": "HALF_DOWN",
      "display_name": "Half Down",
      "labels": {
        "en": "Half Down",
        "gu": "હાફ ડાઉન",
        "hi": "हाफ डाउन"
      },
      "is_active": true,
      "is_default": false,
      "display_order": 2
    }
  ]
}
```

---

## 9.2 Register Option — Internal

```http
POST /internal/v1/configuration/definitions/{key}/options
```

Request:

```json
{
  "value": "HALF_UP",
  "display_name": "Half Up",
  "labels": {
    "en": "Half Up",
    "gu": "હાફ અપ",
    "hi": "हाफ अप"
  },
  "display_order": 1,
  "is_default": true
}
```

---

## 9.3 Update Option

```http
PUT /internal/v1/configuration/definitions/{key}/options/{value}
```

Do not change an option's stored value if existing configuration records depend on it. Deprecate the old value instead.

---

## 9.4 Deactivate Option

```http
DELETE /internal/v1/configuration/definitions/{key}/options/{value}
```

Prefer deactivation over physical deletion.

---

# 10. RESOURCE REGISTRY APIs

Resources represent master-data references owned by other modules.

Examples:

```text
finance.account
organization.warehouse
organization.branch
hr.employee
payment.gateway
sales.price_list
```

---

## 10.1 List Resources

```http
GET /api/v1/configuration/resources
```

Filters:

```text
module
resource_type
search
status
```

---

## 10.2 Get Resource

```http
GET /api/v1/configuration/resources/{resource_code}
```

Example:

```http
GET /api/v1/configuration/resources/finance.account
```

Response:

```json
{
  "success": true,
  "data": {
    "code": "finance.account",
    "module": "finance",
    "resource_type": "REFERENCE",
    "label_field": "name",
    "value_field": "id",
    "search_supported": true,
    "pagination_supported": true,
    "api_contract": {
      "operations": {
        "list": "LIST",
        "search": "SEARCH",
        "get": "GET"
      }
    }
  }
}
```

---

## 10.3 Register Resource — Internal

```http
POST /internal/v1/configuration/resources/register
```

Request:

```json
{
  "code": "finance.account",
  "module": "finance",
  "name": "Finance Account",
  "resource_type": "REFERENCE",
  "label_field": "name",
  "value_field": "id",
  "search_supported": true,
  "pagination_supported": true
}
```

---

# 11. REFERENCE OPTION API FLOW

Definition:

```json
{
  "key": "finance.default_sales_account",
  "data_type": "REFERENCE",
  "reference": {
    "resource": "finance.account"
  },
  "ui_metadata": {
    "component": "ASYNC_SELECT"
  }
}
```

Frontend:

```text
GET definition
    ↓
data_type = REFERENCE
    ↓
resource = finance.account
    ↓
GET resource
    ↓
resolve resource through API Gateway/service registry
    ↓
Finance Account API
```

The configuration database must not contain arbitrary executable URLs.

---

# 12. SETTING VALUE APIs

## 12.1 Get Raw/Effective Setting

```http
GET /api/v1/configuration/settings/{key}
```

Response:

```json
{
  "success": true,
  "data": {
    "key": "finance.invoice.auto_post",
    "value": true,
    "scope": "COMPANY",
    "inherited": true,
    "source": {
      "scope": "COMPANY",
      "tenant_id": "T1",
      "company_id": "C1"
    }
  }
}
```

---

## 12.2 Set Setting

```http
PUT /api/v1/configuration/settings/{key}
```

Request:

```json
{
  "value": true
}
```

The active scope is derived from the authenticated context.

If the caller is setting a broader or narrower scope, the API may accept an explicit scope only after authorization.

Recommended administrative request:

```json
{
  "scope": {
    "type": "BRANCH"
  },
  "value": false,
  "expected_version": 4
}
```

---

## 12.3 Explicit Scope Set — Admin/Internal

```http
PUT /internal/v1/configuration/settings/{key}
```

Request:

```json
{
  "scope": {
    "type": "COMPANY",
    "tenant_id": "T1",
    "company_id": "C1"
  },
  "value": true,
  "expected_version": 4
}
```

The server must validate hierarchy and authorization.

---

## 12.4 Reset Setting

```http
POST /api/v1/configuration/settings/{key}/reset
```

Request:

```json
{
  "scope": "BRANCH"
}
```

Result:

```text
Branch override removed
    ↓
Company value becomes effective
```

Reset should create an audit/change event.

---

## 12.5 Delete Setting Override

```http
DELETE /api/v1/configuration/settings/{key}
```

This should normally remove/deactivate only the current scoped value, not the definition.

---

# 13. BULK SETTING API

```http
PUT /api/v1/configuration/settings/bulk
```

Request:

```json
{
  "items": [
    {
      "key": "finance.invoice.auto_post",
      "value": true
    },
    {
      "key": "finance.invoice.require_approval",
      "value": false
    },
    {
      "key": "finance.tax.rounding_method",
      "value": "HALF_UP"
    }
  ]
}
```

Use transactional behavior where requested.

Recommended:

```json
{
  "transactional": true,
  "items": []
}
```

If transactional and one item fails, roll back all mutations.

---

# 14. EFFECTIVE CONFIGURATION APIs

## 14.1 Complete Effective Configuration

```http
GET /api/v1/configuration/effective
```

Response:

```json
{
  "success": true,
  "data": {
    "finance": {
      "invoice": {
        "auto_post": true,
        "require_approval": false
      },
      "tax": {
        "rounding_method": "HALF_UP"
      }
    },
    "inventory": {
      "stock": {
        "allow_negative": false
      }
    }
  }
}
```

---

## 14.2 Bulk Effective Configuration

```http
POST /api/v1/configuration/effective/bulk
```

Request:

```json
{
  "keys": [
    "finance.invoice.auto_post",
    "finance.invoice.require_approval",
    "finance.tax.rounding_method"
  ]
}
```

Response:

```json
{
  "success": true,
  "data": {
    "finance.invoice.auto_post": {
      "value": true,
      "source_scope": "COMPANY",
      "inherited": true
    },
    "finance.invoice.require_approval": {
      "value": false,
      "source_scope": "BRANCH",
      "inherited": false
    },
    "finance.tax.rounding_method": {
      "value": "HALF_UP",
      "source_scope": "TENANT",
      "inherited": true
    }
  }
}
```

---

# 15. INTERNAL RESOLUTION APIs

## 15.1 Resolve Single Setting

```http
GET /internal/v1/configuration/resolve/{key}
```

Designed for services.

---

## 15.2 Resolve Bulk

```http
POST /internal/v1/configuration/resolve/bulk
```

Request:

```json
{
  "keys": [
    "finance.invoice.auto_post",
    "finance.tax.rounding_method"
  ]
}
```

The service context comes from the service token and active tenant context.

---

# 16. UI METADATA API

## 16.1 Full UI Configuration

```http
GET /api/v1/configuration/ui
```

Returns module/category/definition/options/dependencies metadata plus effective values.

---

## 16.2 Module UI

```http
GET /api/v1/configuration/ui/modules/{module}
```

Example:

```http
GET /api/v1/configuration/ui/modules/finance
```

Response:

```json
{
  "success": true,
  "data": {
    "module": "finance",
    "categories": [
      {
        "code": "tax",
        "name": "Tax",
        "settings": [
          {
            "key": "finance.tax.rounding_method",
            "data_type": "ENUM",
            "value": "HALF_UP",
            "ui": {
              "component": "SELECT",
              "order": 10
            },
            "options": [
              {
                "value": "HALF_UP",
                "label": "Half Up"
              },
              {
                "value": "DOWN",
                "label": "Round Down"
              }
            ]
          }
        ]
      }
    ]
  }
}
```

---

# 17. Localization

The UI API may accept:

```http
GET /api/v1/configuration/ui/modules/finance?locale=gu
```

Labels should be resolved using:

```text
requested locale
    ↓
definition/option localized label
    ↓
English fallback
    ↓
display_name
```

Stored business values remain language-independent.

---

# 18. DEPENDENCY APIs

## List Dependencies

```http
GET /api/v1/configuration/definitions/{key}/dependencies
```

## Register Dependency — Internal

```http
POST /internal/v1/configuration/definitions/{key}/dependencies
```

Request:

```json
{
  "depends_on": "notification.whatsapp.enabled",
  "condition_operator": "EQUALS",
  "condition_value": true,
  "effect_type": "SHOW"
}
```

Supported effects:

```text
SHOW
HIDE
ENABLE
DISABLE
REQUIRE
OPTIONAL
```

---

# 19. POLICY APIs

## List Policies

```http
GET /api/v1/configuration/definitions/{key}/policies
```

## Register Policy — Internal/Admin

```http
POST /internal/v1/configuration/definitions/{key}/policies
```

Example:

```json
{
  "policy_type": "ROLE_RESTRICTION",
  "policy_config": {
    "allowed_roles": [
      "tenant_admin",
      "finance_admin"
    ]
  }
}
```

---

# 20. SECRET APIs

## Get Secret Metadata

```http
GET /api/v1/configuration/secrets/{key}
```

Response:

```json
{
  "success": true,
  "data": {
    "key": "notification.smtp.password",
    "is_secret": true,
    "configured": true,
    "value": null
  }
}
```

## Internal Secret Read

```http
POST /internal/v1/configuration/secrets/{key}/resolve
```

This requires explicit service permission (`X-Internal-Token`). Public JWT GET/rotate never return `_secret_plain` or ciphertext.

Response must only be returned to an authorized trusted service.

---

## Rotate Secret

```http
POST /api/v1/configuration/secrets/{key}/rotate
```

Secrets must never appear in:

```text
logs
audit history
normal API responses
events
Redis normal configuration cache
error messages
```

---

# 21. TEMPLATE APIs

## List Templates

```http
GET /api/v1/configuration/templates
```

## Get Template

```http
GET /api/v1/configuration/templates/{template_code}
```

## Create Template

```http
POST /api/v1/configuration/templates
```

## Update Template

```http
PUT /api/v1/configuration/templates/{template_code}
```

## Apply Template

```http
POST /api/v1/configuration/templates/{template_code}/apply
```

Request:

```json
{
  "scope": {
    "type": "COMPANY",
    "tenant_id": "T1",
    "company_id": "C1"
  },
  "overwrite_existing": false
}
```

Template application must validate every setting.

---

# 22. HISTORY APIs

## Setting History

```http
GET /api/v1/configuration/settings/{key}/history
```

Query:

```text
scope
user_id
from
to
page
page_size
```

Response:

```json
{
  "success": true,
  "data": [
    {
      "action": "UPDATE",
      "old_value": false,
      "new_value": true,
      "scope": "COMPANY",
      "changed_by": "USER_UUID",
      "changed_at": "2026-09-08T10:00:00Z"
    }
  ]
}
```

Secrets must be redacted.

---

# 23. AUDIT APIs

```http
GET /api/v1/configuration/audit
GET /api/v1/configuration/audit/{audit_id}
```

Filters:

```text
module
setting_key
action
actor
tenant
company
branch
from
to
```

Only authorized administrators should access audit data.

---

# 24. RESOURCE REFERENCE APIs

For dynamic UI references, the frontend or gateway can use:

```http
GET /api/v1/configuration/resources/{resource_code}
```

The resource contract can expose logical operations:

```text
LIST
SEARCH
GET
```

Example:

```json
{
  "resource": "finance.account",
  "operations": {
    "list": true,
    "search": true,
    "get": true
  }
}
```

Actual routing is resolved by the API Gateway/service registry.

---

# 25. Dynamic Reference Request

A generic frontend reference component can call the resolved resource endpoint with:

```text
search
page
page_size
sort
filters
```

Example logical request:

```json
{
  "search": "sales",
  "page": 1,
  "page_size": 20
}
```

The returned records should contain at least:

```json
{
  "id": "UUID",
  "label": "Sales Account"
}
```

---

# 26. Configuration Validation API

Optional validation endpoint:

```http
POST /api/v1/configuration/validate
```

Request:

```json
{
  "key": "finance.tax.rounding_method",
  "value": "HALF_UP"
}
```

Response:

```json
{
  "valid": true,
  "errors": []
}
```

This endpoint is useful for UI previews but MUST NOT replace validation during actual mutation.

---

# 27. Preview Effective Configuration

Optional:

```http
POST /api/v1/configuration/preview
```

Request:

```json
{
  "key": "finance.invoice.auto_post",
  "scope": {
    "type": "BRANCH",
    "tenant_id": "T1",
    "company_id": "C1",
    "branch_id": "B1"
  },
  "proposed_value": true
}
```

Response:

```json
{
  "key": "finance.invoice.auto_post",
  "current_value": false,
  "proposed_value": true,
  "effective_after_change": true,
  "dependencies": [],
  "validation_errors": []
}
```

No database mutation occurs.

---

# 28. Import APIs

Optional:

```http
POST /api/v1/configuration/import
```

Use for controlled migrations.

Request:

```json
{
  "mode": "VALIDATE_ONLY",
  "items": []
}
```

Supported modes:

```text
VALIDATE_ONLY
CREATE
UPSERT
```

Secrets must be excluded from normal import/export.

---

# 29. Export API

```http
GET /api/v1/configuration/export
```

Filters:

```text
module
category
scope
```

Export format should contain:

```text
key
scope
value
definition version
```

Secret values must not be exported by normal API.

---

# 30. Cache APIs

Cache invalidation should normally happen automatically through events.

Internal administrative endpoint:

```http
POST /internal/v1/configuration/cache/invalidate
```

Request:

```json
{
  "tenant_id": "T1",
  "keys": [
    "finance.invoice.auto_post"
  ]
}
```

This endpoint must be highly restricted.

---

# 31. Health APIs

```http
GET /health
GET /health/live
GET /health/ready
GET /version
GET /metrics
```

Readiness should verify required dependencies:

```text
PostgreSQL
Redis
message/outbox infrastructure
```

---

# 32. Complete Example — Finance Registration

Finance starts and calls:

```http
POST /internal/v1/configuration/modules/register
```

Then:

```http
POST /internal/v1/configuration/categories/register
```

Then:

```http
POST /internal/v1/configuration/definitions/register
```

for:

```text
finance.invoice.auto_post
```

Then:

```http
POST /internal/v1/configuration/definitions/register
```

for:

```text
finance.tax.rounding_method
```

Then:

```http
POST /internal/v1/configuration/definitions/finance.tax.rounding_method/options
```

for each ENUM option.

Then:

```http
POST /internal/v1/configuration/resources/register
```

for:

```text
finance.account
```

The module registration process is idempotent.

---

# 33. Complete Example — User Changes Setting

Frontend loads:

```http
GET /api/v1/configuration/ui/modules/finance
```

It renders:

```text
Finance
  Tax
    Tax Rounding Method
      [ Half Up ▼ ]
```

User selects:

```text
Round Down
```

Frontend sends:

```http
PUT /api/v1/configuration/settings/finance.tax.rounding_method
```

```json
{
  "value": "DOWN",
  "expected_version": 4
}
```

Server:

```text
Authenticate
 ↓
Resolve tenant/company/branch
 ↓
Authorize
 ↓
Load definition
 ↓
Validate ENUM option
 ↓
Validate scope
 ↓
Validate dependencies/policies
 ↓
Update value
 ↓
Create change record
 ↓
Create outbox event
 ↓
Commit
```

Worker:

```text
Outbox
 ↓
Event Bus
 ↓
Redis invalidation
 ↓
Finance consumer
```

---

# 34. Error Contract

Recommended HTTP mapping:

```text
400 CONFIGURATION_INVALID_VALUE
400 CONFIGURATION_INVALID_TYPE
400 CONFIGURATION_SCOPE_NOT_ALLOWED
400 CONFIGURATION_DEPENDENCY_FAILED
401 CONFIGURATION_UNAUTHENTICATED
403 CONFIGURATION_UNAUTHORIZED
403 CONFIGURATION_SECRET_ACCESS_DENIED
404 CONFIGURATION_NOT_FOUND
404 CONFIGURATION_DEFINITION_NOT_FOUND
404 CONFIGURATION_RESOURCE_NOT_FOUND
409 CONFIGURATION_VERSION_CONFLICT
409 CONFIGURATION_DUPLICATE
422 CONFIGURATION_VALIDATION_FAILED
429 CONFIGURATION_RATE_LIMITED
500 CONFIGURATION_INTERNAL_ERROR
503 CONFIGURATION_DEPENDENCY_UNAVAILABLE
```

---

# 35. Pagination

List APIs support:

```text
page
page_size
sort
order
search
```

Recommended limits:

```text
default page_size = 25
maximum page_size = 100
```

Large exports should use asynchronous jobs.

---

# 36. Filtering

Support standard filters:

```text
status
module
category
data_type
scope
is_editable
is_secret
created_at
updated_at
```

Do not allow arbitrary SQL expressions through query parameters.

---

# 37. Idempotency

Registration APIs:

```text
module register
category register
definition register
option register
resource register
```

must be idempotent.

Mutation APIs with retry risk should accept:

```http
Idempotency-Key
```

The server should persist enough information to return the original result for a repeated request.

---

# 38. Optimistic Concurrency

Updates should support the integer `version` counter, sent either as a header or a body field (both carry the same value):

```http
If-Match: 4
```

or:

```json
{
  "expected_version": 4
}
```

If the stored `version` has advanced past the supplied value:

```text
409 CONFIGURATION_VERSION_CONFLICT
```

`row_version` (UUID) is an internal change token and is never the `If-Match` value.

---

# 39. Rate Limiting

Recommended:

```text
Public read APIs:
high limit

Public write APIs:
lower limit

Secret APIs:
very low limit

Internal resolution APIs:
service-specific quotas
```

Do not expose secret endpoints without strong authorization and rate limits.

---

# 40. API Design Rule for Dynamic APIs

The Configuration Platform should describe:

```text
WHAT resource is required
```

not:

```text
EXECUTE THIS URL
```

Correct:

```json
{
  "reference": {
    "resource": "finance.account"
  }
}
```

Avoid:

```json
{
  "api_url": "http://finance-service:8005/accounts"
}
```

The API Gateway/service registry owns service routing.

---

# 41. API Design Rule for Business Reactions

Do not configure:

```json
{
  "on_change": {
    "method": "POST",
    "url": "/finance/recalculate"
  }
}
```

Instead:

```text
configuration.setting.updated
        ↓
event bus
        ↓
Finance
        ↓
Finance decides business reaction
```

This preserves bounded-context ownership.

---

# 42. Recommended API Router Structure

```text
platforms/p03_configuration/infrastructure/http/
│
├── api_v1.py
├── admin_api.py
├── tenant_api.py
├── routers/
│   ├── modules.py
│   ├── categories.py
│   ├── definitions.py
│   ├── options.py
│   ├── settings.py
│   ├── effective.py
│   ├── resources.py
│   ├── templates.py
│   ├── secrets.py
│   ├── history.py
│   ├── audit.py
│   ├── dependencies.py
│   ├── policies.py
│   ├── ui.py
│   ├── cache.py
│   ├── resolve.py          # internal
│   └── health.py
├── schemas/
├── dependencies/
└── exception_handlers.py
```

---

# 43. Pydantic Request Models

Example:

```python
class SetConfigurationRequest(BaseModel):
    value: Any
    expected_version: int | None = None
```

Scope:

```python
class ConfigurationScopeRequest(BaseModel):
    type: Literal[
        "SYSTEM",
        "TENANT",
        "COMPANY",
        "BRANCH",
        "ROLE",
        "USER",
    ]
    tenant_id: UUID | None = None
    company_id: UUID | None = None
    branch_id: UUID | None = None
    role_id: UUID | None = None
    user_id: UUID | None = None
```

Internal APIs may expose explicit scope fields, but authorization and hierarchy validation are mandatory.

---

# 44. API Versioning

Use:

```text
/api/v1/configuration
/internal/v1/configuration
```

Breaking changes use:

```text
v2
```

Do not silently change the meaning of a response field.

Module registration should include definition version.

---

# 45. API Security Checklist

Every endpoint must answer:

```text
Who is calling?
Which tenant?
Which company?
Which branch?
Which scope?
Which setting?
Is this scope allowed?
Is the caller authorized?
Is the value valid?
Is the reference valid?
Is the definition active?
Is the definition editable?
Is there a dependency?
Is there a policy?
Is concurrency valid?
```

---

# 46. Final API Contract

The most important APIs for normal ERP operation are:

```text
GET  /api/v1/configuration/ui/modules/{module}

GET  /api/v1/configuration/settings/{key}

PUT  /api/v1/configuration/settings/{key}

POST /api/v1/configuration/settings/{key}/reset

GET  /api/v1/configuration/effective

POST /api/v1/configuration/effective/bulk

GET  /api/v1/configuration/resources/{resource_code}
```

For module integration:

```text
POST /internal/v1/configuration/modules/register

POST /internal/v1/configuration/categories/register

POST /internal/v1/configuration/definitions/register

POST /internal/v1/configuration/definitions/{key}/options

POST /internal/v1/configuration/resources/register

GET  /internal/v1/configuration/resolve/{key}

POST /internal/v1/configuration/resolve/bulk
```

This is the stable contract that Finance, Inventory, HR, Sales, CRM, and future JeslotERP services should consume.

---

# 47. Final Rule

The complete dynamic chain is:

```text
MODULE
  |
  | registers
  v
DEFINITION
  |
  +--> DATA TYPE
  |
  +--> ENUM OPTIONS
  |
  +--> UI METADATA
  |
  +--> REFERENCE RESOURCE
  |
  +--> VALIDATION
  |
  +--> DEPENDENCIES
  |
  +--> POLICIES
  |
  v
CONFIGURATION VALUE
  |
  v
SCOPE RESOLUTION
  |
  v
EFFECTIVE VALUE
  |
  +--> FRONTEND renders dynamically
  |
  +--> MODULE consumes through SDK/API
  |
  v
CHANGE EVENT
  |
  v
MODULE decides business reaction
```

The Configuration Platform must remain a configuration and metadata platform, not an arbitrary API execution engine.
