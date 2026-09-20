# JeslotERP Configuration Platform — Developer Guide

**Version:** 1.3  
**Last reviewed:** 2026-09-08  
**Status:** **SoR-Live** — secrets are ciphertext (not `_secret_plain`). Not Production.

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial developer guide baseline. |
| 1.1 | 2026-09-08 | Optimistic concurrency clarified as the integer `version` via `If-Match` (§45), matching the schema/API; module registry field `version → module_version` (§10). |
| 1.2 | 2026-09-08 | Aligned folder structure with ModulePlugin + `infrastructure/http`; reconciled public/internal API inventories with Complete API; unified permission catalog; fixed secret resolve verb; added IAM/ORG dependencies, RLS GUCs, outbox ownership, seed/bootstrap, and consumer compatibility aliases. |
| 1.3 | 2026-09-08 | Froze package as `platforms.p03_configuration` (aligned with docs folder `03_configuration`). Business Partner remains a separate platform and must not reuse this package name. |
| 1.4 | 2026-09-12 | PROD-KERN-003: §31–32 inventory matches shipped routers (API.md authoritative). |

---

# 0. Repository Integration Contract

This document is the implementer contract for **this repository**.

| Item | Frozen value |
|---|---|
| Docs folder | `docs/platforms/03_configuration/` |
| Platform package | `platforms.p03_configuration` |
| `ModulePlugin.name` | `p03_configuration` |
| Module dependencies | `["p01_identity", "p02_organization"]` |
| PostgreSQL schema | `configuration` |
| Public API | `/api/v1/configuration` |
| Internal API | `/internal/v1/configuration` |
| HTTP layer | `infrastructure/http/` (not `interfaces/http/`) |
| Consumer admin router | `infrastructure.http.admin_api` |
| Consumer tenant router | `infrastructure.http.tenant_api` |
| Pagination legacy aliases | `ConfigurationCoreSetting` / `ConfigurationCoreSettingValue` in `models/setting.py` (see §0.3) |

**Do not** place Configuration on `p04_*`. Package number **03** is Configuration. Existing `p03_business_partner` import sites must be renumbered when that platform is restored (they must not collide with `platforms.p03_configuration`). Canonical map: [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md).

### 0.1 Bounded-context dependencies

- **IAM (`p01_identity`)** owns authentication, sessions, permissions, and role assignment. Configuration consumes JWT context and IAM permission codes; it must not implement login.
- **ORG (`p02_organization`)** owns tenant/company/branch hierarchy. Configuration validates hierarchy through an ORG gateway/port — never by importing ORG ORM models.
- **ORG tenant settings/branding** remain ORG-owned operational metadata. Runtime ERP settings (page size, feature toggles, module policies) belong in Configuration. Do not duplicate ORG structure into Configuration tables.

### 0.2 Source-of-truth precedence

1. **Schema DDL** in `CONFIGURATION_SCHEMA.md` — tables/columns/constraints.
2. **Complete API** document — HTTP paths, verbs, envelopes, permissions.
3. **This Developer Guide** — architecture, DoD, coding rules.

When inventories diverge, Complete API wins for HTTP; Schema wins for DDL.

### 0.3 Consumer compatibility aliases

Existing code (`shared/pagination/dependencies.py`) imports:

```python
from platforms.p03_configuration.infrastructure.persistence.models.setting import (
    ConfigurationCoreSetting,
    ConfigurationCoreSettingValue,
)
```

These are **compatibility aliases** over the production tables:

| Alias | Maps to |
|---|---|
| `ConfigurationCoreSetting` | `configuration_definition` (key exposed as `setting_key`) |
| `ConfigurationCoreSettingValue` | `configuration_value` |

New code must prefer the canonical model names. Aliases remain until pagination is migrated.

### 0.4 Seed / bootstrap

First migration must seed:

- Permission catalog into IAM (`configuration.*` codes from Complete API §5).
- System module `system` with definition `system.ui.default_page_rows` (key also readable as `default_page_rows` via alias) defaulting to `10`.

### 0.5 Outbox ownership

Configuration owns `configuration.outbox_event` (same shape as ORG/IAM outbox). Event `event_type` strings use dotted bus names (`configuration.setting.updated`). Audit table `event_type` values remain uppercase domain codes (`SETTING_UPDATED`) and must not be mixed.

### 0.6 RLS GUCs

Tenant-scoped tables use PostgreSQL RLS with session GUCs (mirroring ORG):

```text
app.tenant_id
app.company_id
app.branch_id
app.user_id
app.rls_bypass   -- service/admin only
```

Application authz remains mandatory; RLS is defense-in-depth.

## 1. Purpose

This guide defines how developers must implement, register, consume, secure, cache, test, and operate the JeslotERP Configuration Platform.

The Configuration Platform is a centralized bounded context responsible for configuration definitions, values, scope resolution, validation, metadata-driven UI behavior, resource references, secrets, audit history, caching, and configuration events.

Core rule:

> Modules own the meaning of their configuration. The Configuration Platform owns configuration storage and infrastructure behavior.

---

# 2. What the Configuration Platform Solves

Without a central platform:

```text
Finance
  finance_settings

Inventory
  inventory_settings

HR
  hr_settings

Sales
  sales_settings
```

This creates duplicated authorization, validation, caching, audit, and UI logic.

With the Configuration Platform:

```text
                    Configuration Platform
                              |
          ┌───────────────────┼───────────────────┐
          |                   |                   |
      Definitions           Values             Metadata
          |                   |                   |
          └───────────────────┼───────────────────┘
                              |
          ┌───────────────────┼───────────────────┐
          |                   |                   |
       Finance            Inventory              HR
```

Every module uses the same configuration infrastructure.

---

# 3. Architectural Principles

The implementation MUST follow:

1. DDD
2. Hexagonal Architecture
3. Clean Architecture
4. CQRS
5. Multi-tenant isolation
6. Token-bound request context
7. Transactional Outbox
8. Redis cache-aside
9. Optimistic concurrency
10. Immutable audit/change history
11. Metadata-driven frontend
12. Internal service APIs
13. Public administration APIs
14. Module-owned configuration definitions
15. PostgreSQL as source of truth

---

# 4. Bounded Context Ownership

The Configuration bounded context owns:

```text
configuration_module
configuration_category
configuration_definition
configuration_option
configuration_resource
configuration_value
configuration_dependency
configuration_policy
configuration_secret
configuration_template
configuration_template_value
configuration_change
configuration_audit
```

Finance owns:

```text
invoice
journal
account
tax
payment
```

Inventory owns:

```text
item
warehouse
stock
stock transaction
```

Configuration may reference these resources but must not own them.

---

# 5. Golden Rule

Never allow:

```python
from platforms.p03_configuration.infrastructure.persistence.models import ConfigurationValue
```

inside Finance domain/application code.

Never allow:

```sql
UPDATE configuration.configuration_value
```

from another module.

Use:

```python
await configuration.get(...)
await configuration.set(...)
await configuration.register(...)
```

through a port/SDK (`platforms.p03_configuration.application` / internal resolve API).

---

# 6. System Architecture

Recommended initial architecture:

```text
React
  |
API Gateway
  |
FastAPI Application
  |
Configuration Bounded Context
  |
  +-- Domain
  +-- Application
  +-- Infrastructure
  +-- HTTP
  |
PostgreSQL
Redis
Celery
Outbox
```

Later extraction:

```text
Finance Service
       |
       v
Configuration SDK
       |
       v
Configuration Service
       |
       +-- PostgreSQL
       +-- Redis
       +-- Event Bus
```

Application code should not depend on whether configuration is in-process or remote.

---

# 7. Technology

Recommended:

```text
Python 3.12+
FastAPI
Pydantic v2
SQLAlchemy 2.x Async
PostgreSQL 16+
asyncpg
Redis 7+
Celery
Alembic
uv
```

Domain code must not directly depend on:

```text
FastAPI
SQLAlchemy
Redis
Celery
JWT libraries
Pydantic
PostgreSQL
HTTP clients
```

Framework-specific implementation belongs at the infrastructure/interface boundary.

---

# 8. Folder Structure

Align with `platforms/p01_identity` and `platforms/p02_organization`:

```text
platforms/
└── p03_configuration/
    ├── module.py                          # ModulePlugin entry
    ├── domain/
    │   ├── aggregates/
    │   ├── value_objects/
    │   │   ├── configuration_key.py
    │   │   ├── configuration_scope.py
    │   │   └── configuration_type.py
    │   ├── services/
    │   │   ├── configuration_resolver.py  # pure resolution rules
    │   │   ├── configuration_validator.py
    │   │   └── dependency_evaluator.py
    │   ├── exceptions.py
    │   └── events/
    │       # Domain event classes; bus event_type strings live in application/messaging
    │
    ├── application/
    │   ├── commands/
    │   ├── queries/
    │   ├── services/
    │   ├── ports/
    │   ├── permissions/
    │   │   └── catalog.py                 # configuration.* IAM seed codes
    │   └── sdk/                           # in-process client for other platforms
    │
    ├── infrastructure/
    │   ├── module.py                      # re-export Organization-style
    │   ├── persistence/
    │   │   ├── models/
    │   │   │   ├── setting.py             # ConfigurationCoreSetting aliases
    │   │   │   └── ...
    │   │   ├── repositories/
    │   │   ├── services/
    │   │   └── rls.py
    │   ├── cache/
    │   ├── encryption/
    │   ├── messaging/
    │   │   └── outbox/
    │   ├── http/
    │   │   ├── api_v1.py                  # mounts public + internal routers
    │   │   ├── admin_api.py               # SuperAdmin consumer router
    │   │   ├── tenant_api.py              # Tenant consumer router
    │   │   ├── routers/
    │   │   ├── schemas/
    │   │   ├── dependencies/
    │   │   └── exception_handlers.py
    │   └── gateways/
    │       └── organization/              # hierarchy validation port adapter
    │
    └── tests/
```

`ModulePlugin` must expose `name="p03_configuration"`, `dependencies=["p01_identity", "p02_organization"]`, and register routers under `/api/v1/configuration` + `/internal/v1/configuration`.

---

# 9. Configuration Key Standard

Keys MUST be globally unique.

Recommended:

```text
{module}.{category}.{name}
```

Examples:

```text
finance.invoice.auto_post
finance.invoice.require_approval
finance.tax.rounding_method
inventory.stock.allow_negative
inventory.warehouse.default
sales.invoice.numbering_method
notification.whatsapp.enabled
```

Use lowercase snake_case segments.

Do not use:

```text
AutoPost
Finance.AutoPost
finance_invoice_auto_post
```

---

# 10. Module Registration

A module registers itself.

Example:

```python
class FinanceConfigurationRegistry:

    module = {
        "code": "finance",
        "name": "Finance",
        "module_version": "1.0.0",
        "service_name": "finance-service",
    }
```

Startup:

```python
await configuration.register_module(...)
```

Registration must be idempotent.

Repeated deployment must not create duplicates.

---

# 11. Category Registration

Example:

```python
categories = [
    {
        "module": "finance",
        "code": "general",
        "name": "General",
    },
    {
        "module": "finance",
        "code": "invoice",
        "name": "Invoice",
    },
    {
        "module": "finance",
        "code": "tax",
        "name": "Tax",
    },
]
```

---

# 12. Definition Registration

Example:

```python
{
    "key": "finance.invoice.auto_post",
    "category": "invoice",
    "display_name": "Auto Post Invoice",
    "description": "Automatically post an invoice after submission.",
    "data_type": "BOOLEAN",
    "default_value": False,
    "allowed_scopes": [
        "TENANT",
        "COMPANY",
        "BRANCH"
    ],
    "ui_metadata": {
        "component": "SWITCH"
    }
}
```

---

# 13. ENUM Settings

Definition:

```python
{
    "key": "finance.tax.rounding_method",
    "data_type": "ENUM",
    "default_value": "HALF_UP",
    "allowed_scopes": [
        "TENANT",
        "COMPANY",
        "BRANCH"
    ],
    "ui_metadata": {
        "component": "SELECT"
    }
}
```

Options:

```python
[
    {
        "value": "HALF_UP",
        "display_name": "Half Up",
    },
    {
        "value": "HALF_DOWN",
        "display_name": "Half Down",
    },
    {
        "value": "UP",
        "display_name": "Round Up",
    },
    {
        "value": "DOWN",
        "display_name": "Round Down",
    },
]
```

The database stores:

```text
HALF_UP
```

The UI displays:

```text
Half Up
```

---

# 14. Localization

Option labels may contain:

```json
{
  "en": "Half Up",
  "gu": "હાફ અપ",
  "hi": "हाफ अप"
}
```

The stored value remains:

```text
HALF_UP
```

Localization must never change the business value.

---

# 15. REFERENCE Settings

Use `REFERENCE` when the setting points to master data.

Example:

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

Stored value:

```text
ACCOUNT_UUID
```

Not:

```text
Cash Account
```

---

# 16. MULTI_REFERENCE

Example:

```json
{
  "key": "finance.allowed_tax_accounts",
  "data_type": "MULTI_REFERENCE",
  "reference": {
    "resource": "finance.account"
  },
  "ui_metadata": {
    "component": "ASYNC_MULTI_SELECT"
  }
}
```

Stored value:

```json
[
    "ACCOUNT_UUID_1",
    "ACCOUNT_UUID_2"
]
```

---

# 17. Resource Registry

The resource registry prevents settings from hardcoding module-specific API URLs.

Example resource:

```text
finance.account
```

Definition:

```json
{
  "code": "finance.account",
  "module": "finance",
  "resource_type": "REFERENCE",
  "label_field": "name",
  "value_field": "id"
}
```

The gateway/service registry resolves the actual endpoint.

Do not store:

```text
http://finance-service:8005/api/v1/accounts
```

inside configuration values.

---

# 18. Dynamic Reference Flow

```text
Frontend
   |
   | sees data_type=REFERENCE
   |
   v
resource=finance.account
   |
   v
Resource Registry
   |
   v
API Gateway / Service Discovery
   |
   v
Finance Account API
```

The frontend should not contain:

```typescript
if (key === "finance.default_sales_account") {
    ...
}
```

---

# 19. UI Metadata

The definition can specify:

```json
{
  "ui_metadata": {
    "component": "SELECT",
    "order": 10,
    "group": "Tax",
    "help_text": "Determines how tax amounts are rounded"
  }
}
```

Supported generic components:

```text
TEXT
TEXTAREA
NUMBER
DECIMAL
SWITCH
CHECKBOX
SELECT
MULTI_SELECT
DATE
DATETIME
TIME
JSON_EDITOR
ASYNC_SELECT
ASYNC_MULTI_SELECT
SECRET
```

The frontend should choose the component using metadata.

---

# 20. Metadata-Driven Frontend

Frontend receives:

```json
{
  "key": "finance.invoice.auto_post",
  "data_type": "BOOLEAN",
  "ui_metadata": {
    "component": "SWITCH"
  },
  "value": true
}
```

Then:

```text
BOOLEAN + SWITCH
       ↓
Switch component
```

For ENUM:

```text
ENUM + SELECT
       ↓
Select component
```

For REFERENCE:

```text
REFERENCE + ASYNC_SELECT
       ↓
Reference component
```

---

# 21. Conditional Settings

Example:

```text
notification.whatsapp.enabled
```

controls:

```text
notification.whatsapp.provider
notification.whatsapp.sender
notification.whatsapp.template
```

Dependency:

```json
{
  "definition": "notification.whatsapp.provider",
  "depends_on": "notification.whatsapp.enabled",
  "operator": "EQUALS",
  "value": true,
  "effect": "SHOW"
}
```

The frontend evaluates dependency metadata.

Backend must still validate dependencies.

Frontend behavior is never a security boundary.

---

# 22. Validation

Validation belongs to the Configuration Platform.

Example:

```json
{
  "data_type": "INTEGER",
  "validation_rules": {
    "min": 1,
    "max": 20
  }
}
```

ENUM:

```json
{
  "data_type": "ENUM"
}
```

The submitted value must exist in active options.

STRING:

```json
{
  "validation_rules": {
    "min_length": 3,
    "max_length": 100
  }
}
```

JSON:

```json
{
  "validation_rules": {
    "schema": {}
  }
}
```

---

# 23. Scope Resolution

Given:

```text
key
user_id
role_id
tenant_id
company_id
branch_id
current_time
```

resolution order:

```text
USER
 ↓
ROLE
 ↓
BRANCH
 ↓
COMPANY
 ↓
TENANT
 ↓
SYSTEM
 ↓
DEFINITION DEFAULT
```

Only scopes allowed by the definition participate.

---

# 24. Scope Example

Definition:

```text
finance.invoice.auto_post
```

System:

```text
false
```

Tenant:

```text
false
```

Company:

```text
true
```

Branch:

```text
no value
```

User:

```text
no value
```

Effective:

```text
true
```

Source:

```text
COMPANY
```

---

# 25. Inheritance Response

API:

```http
GET /api/v1/configuration/settings/finance.invoice.auto_post
```

Response:

```json
{
  "key": "finance.invoice.auto_post",
  "value": true,
  "inherited": true,
  "source": {
    "scope": "COMPANY",
    "tenant_id": "T1",
    "company_id": "C1"
  }
}
```

This allows the frontend to display:

```text
Value: Enabled
Source: Company
Inherited: Yes
```

---

# 26. Override and Reset

User can override:

```text
Company value = true
Branch value = false
```

Reset branch:

```http
POST /api/v1/configuration/settings/{key}/reset
```

After reset:

```text
Branch has no value
       ↓
Company value applies
```

Reset must not delete definition metadata.

---

# 27. Effective Configuration API

UI should be able to load a complete effective configuration:

```http
GET /api/v1/configuration/effective
```

Example:

```json
{
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
```

---

# 28. Bulk Resolution

Services should avoid repeated network calls.

Use:

```http
POST /internal/v1/configuration/resolve/bulk
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
  "data": {
    "finance.invoice.auto_post": true,
    "finance.invoice.require_approval": false,
    "finance.tax.rounding_method": "HALF_UP"
  }
}
```

---

# 29. SDK

Recommended service API:

```python
value = await config.get(
    "finance.invoice.auto_post"
)
```

Bulk:

```python
settings = await config.get_many([
    "finance.invoice.auto_post",
    "finance.invoice.require_approval",
])
```

Write:

```python
await config.set(
    key="finance.invoice.auto_post",
    value=True,
)
```

Registration:

```python
await config.register(
    module="finance",
    definitions=FINANCE_CONFIGURATION,
)
```

---

# 30. Public vs Internal APIs

Public:

```text
/api/v1/configuration
```

Used by:

```text
React
Tenant administrators
Authorized ERP users
```

Internal:

```text
/internal/v1/configuration
```

Used by:

```text
Finance service
Inventory service
HR service
CRM service
Platform services
```

Internal APIs require service authentication and authorization.

---

# 31. Public API Inventory

Canonical inventory (Complete API document is authoritative). Summary:

```text
GET    /api/v1/configuration/modules
GET    /api/v1/configuration/modules/{module_code}
GET    /api/v1/configuration/categories
GET    /api/v1/configuration/categories/{category_code}
PUT    /api/v1/configuration/categories/{category_code}

GET    /api/v1/configuration/definitions
GET    /api/v1/configuration/definitions/{key}

GET    /api/v1/configuration/definitions/{key}/options
PUT    /api/v1/configuration/definitions/{key}/options/{value}
DELETE /api/v1/configuration/definitions/{key}/options/{value}

GET    /api/v1/configuration/resources
GET    /api/v1/configuration/resources/{resource_code}

GET    /api/v1/configuration/settings/{key}
PUT    /api/v1/configuration/settings/{key}
PUT    /api/v1/configuration/settings/bulk
DELETE /api/v1/configuration/settings/{key}
POST   /api/v1/configuration/settings/{key}/reset
GET    /api/v1/configuration/settings/{key}/history

GET    /api/v1/configuration/effective
POST   /api/v1/configuration/effective/bulk
POST   /api/v1/configuration/preview

GET    /api/v1/configuration/scopes

GET    /api/v1/configuration/ui
GET    /api/v1/configuration/ui/modules/{module}

GET    /api/v1/configuration/definitions/{key}/dependencies
GET    /api/v1/configuration/definitions/{key}/policies

GET    /api/v1/configuration/secrets/{key}
POST   /api/v1/configuration/secrets/{key}/rotate

GET    /api/v1/configuration/templates
GET    /api/v1/configuration/templates/{template_code}
POST   /api/v1/configuration/templates
PUT    /api/v1/configuration/templates/{template_code}
POST   /api/v1/configuration/templates/{template_code}/apply

GET    /api/v1/configuration/audit
GET    /api/v1/configuration/audit/{audit_id}
POST   /api/v1/configuration/validate
POST   /api/v1/configuration/import
GET    /api/v1/configuration/export
```

---

# 32. Internal API Inventory

```text
POST /internal/v1/configuration/modules/register
PUT  /internal/v1/configuration/modules/{module_code}

POST /internal/v1/configuration/categories/register

POST /internal/v1/configuration/definitions/register
PUT  /internal/v1/configuration/definitions/{key}
DELETE /internal/v1/configuration/definitions/{key}
POST /internal/v1/configuration/definitions/{key}/options

POST /internal/v1/configuration/resources/register

PUT  /internal/v1/configuration/settings/{key}

GET  /internal/v1/configuration/resolve/{key}
POST /internal/v1/configuration/resolve/bulk

POST /internal/v1/configuration/secrets/{key}/resolve

POST /internal/v1/configuration/definitions/{key}/dependencies
POST /internal/v1/configuration/definitions/{key}/policies

POST /internal/v1/configuration/cache/invalidate
```

---

# 33. Authentication and Authorization

Every request requires authenticated context.

The service derives:

```text
user_id
tenant_id
company_id
branch_id
session_id
scopes
roles
permissions
```

from the trusted authentication context.

Canonical permission catalog (Complete API §5):

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

# 34. Service-to-Service Authentication

Internal module requests should use service identity.

Example:

```text
finance-service
     |
     | service token
     v
configuration-service
```

The Configuration Platform validates:

```text
issuer
audience
signature
service identity
permissions
tenant context
```

A service must only access permitted modules/tenants.

---

# 35. Tenant Isolation

All tenant-scoped operations must verify:

```text
request.tenant_id
=
target.tenant_id
```

Hierarchy:

```text
Tenant
  ↓
Company
  ↓
Branch
```

The service must verify that:

```text
company belongs to tenant
branch belongs to company
```

Never trust IDs sent by the browser.

---

# 36. Context Switching

When the user changes:

```text
Company
Branch
```

IAM issues a new context-bound access token.

Configuration automatically resolves using the new context.

Frontend does not send:

```text
X-Tenant-ID
X-Company-ID
X-Branch-ID
```

as security headers.

---

# 37. Secrets

Secret examples:

```text
notification.smtp.password
notification.whatsapp.api_secret
payment.razorpay.secret
gst.api_secret
```

Use:

```text
AES-256-GCM
+
KMS / key manager
+
key versioning
```

Never log secrets.

Never include decrypted secrets in standard configuration responses.

---

# 38. Secret Response

Normal API:

```json
{
  "key": "notification.smtp.password",
  "is_secret": true,
  "configured": true,
  "value": null
}
```

A dedicated authorized internal secret operation may retrieve the decrypted value.

Secret access must be audited.

---

# 39. Caching

Configuration resolution is read-heavy.

Use Redis cache-aside:

```text
Service
   ↓
Redis
   |
   +-- HIT → return
   |
   +-- MISS
          ↓
     PostgreSQL
          ↓
        Redis
          ↓
        return
```

PostgreSQL remains the source of truth.

---

# 40. Cache Key

Recommended:

```text
config:v1:{tenant_id}:{company_id}:{branch_id}:{user_id}:{key}
```

For bulk effective configuration:

```text
config-effective:v1:{tenant_id}:{company_id}:{branch_id}:{user_id}
```

If role scope is used, include applicable role context.

---

# 41. Cache Invalidation

When a setting changes:

```text
DB transaction
    ↓
configuration_value
    ↓
configuration_change
    ↓
outbox_event
    ↓
COMMIT
    ↓
worker
    ↓
publish event
    ↓
invalidate Redis
```

Do not rely only on TTL for immediate configuration changes.

---

# 42. Configuration Events

Recommended events:

```text
configuration.module.registered
configuration.definition.created
configuration.definition.updated
configuration.setting.created
configuration.setting.updated
configuration.setting.reset
configuration.setting.deleted
configuration.secret.rotated
configuration.template.applied
```

Event payload:

```json
{
  "event_id": "EVENT_UUID",
  "event_type": "configuration.setting.updated",
  "version": 1,
  "occurred_at": "2026-09-08T10:00:00Z",

  "tenant_id": "TENANT_UUID",
  "company_id": "COMPANY_UUID",
  "branch_id": "BRANCH_UUID",

  "setting_key": "finance.invoice.auto_post",

  "old_value": false,
  "new_value": true,

  "actor_user_id": "USER_UUID",
  "request_id": "REQUEST_UUID",
  "correlation_id": "CORRELATION_UUID"
}
```

Never put plaintext secrets into events.

---

# 43. Event Consumer Rule

Finance may consume:

```text
configuration.setting.updated
```

and decide:

```python
if event.setting_key == "finance.invoice.auto_post":
    await refresh_invoice_configuration()
```

Configuration itself does not call:

```text
finance/recalculate
```

based on a stored URL.

This keeps bounded contexts decoupled.

---

# 44. Transaction Boundary

A setting update must be atomic:

```text
BEGIN

validate definition
validate value
validate scope

UPDATE configuration_value

INSERT configuration_change

INSERT outbox_event

COMMIT
```

If any step fails:

```text
ROLLBACK
```

No event should be published for a transaction that did not commit.

---

# 45. Optimistic Concurrency

Clients send the integer optimistic-lock `version` (the schema's `version` counter — the same convention used across every JeslotERP platform), either as an `If-Match` header or an `expected_version` body field:

```text
If-Match: 4
```

Example body form:

```json
{
  "value": true,
  "expected_version": 4
}
```

If the stored `version` is now:

```text
5
```

return:

```text
409 CONFIGURATION_VERSION_CONFLICT
```

(`row_version`, the UUID column, is an internal change token — not the `If-Match` value.)

Never silently overwrite another administrator's change.

---

# 46. Error Codes

Recommended:

```text
CONFIGURATION_NOT_FOUND
CONFIGURATION_DEFINITION_NOT_FOUND
CONFIGURATION_INVALID_TYPE
CONFIGURATION_INVALID_VALUE
CONFIGURATION_SCOPE_NOT_ALLOWED
CONFIGURATION_UNAUTHORIZED
CONFIGURATION_VERSION_CONFLICT
CONFIGURATION_DEPENDENCY_FAILED
CONFIGURATION_RESOURCE_NOT_FOUND
CONFIGURATION_SECRET_ACCESS_DENIED
CONFIGURATION_INVALID_REFERENCE
CONFIGURATION_MODULE_NOT_REGISTERED
```

---

# 47. Query vs Command

Commands:

```text
RegisterModule
RegisterCategory
RegisterDefinition
SetConfigurationValue
ResetConfigurationValue
RegisterOption
RegisterResource
ApplyTemplate
RotateSecret
```

Queries:

```text
GetDefinition
GetSetting
GetEffectiveSetting
GetEffectiveConfiguration
ListDefinitions
ListModules
ListCategories
ListResources
GetHistory
```

Commands mutate state.

Queries must not mutate state.

---

# 48. Repository Ports

Domain/application should define ports:

```python
class ConfigurationDefinitionRepository(Protocol):
    async def get_by_key(
        self,
        key: ConfigurationKey,
    ) -> ConfigurationDefinition | None:
        ...

class ConfigurationValueRepository(Protocol):
    async def get_effective(
        self,
        key: ConfigurationKey,
        context: ConfigurationContext,
    ) -> ConfigurationValue | None:
        ...

    async def save(
        self,
        value: ConfigurationValue,
    ) -> None:
        ...
```

Infrastructure implements them.

---

# 49. Query Optimization

For effective configuration reads:

```text
Do not load ORM object graphs unnecessarily.
```

Use:

```text
SQLAlchemy Core
asyncpg
optimized SQL
Redis
```

where appropriate.

Read models can be optimized independently from write models.

---

# 50. Effective Resolution SQL Strategy

Resolution should rank candidates by scope specificity.

Conceptually:

```text
USER       priority 1
ROLE       priority 2
BRANCH     priority 3
COMPANY    priority 4
TENANT     priority 5
SYSTEM     priority 6
DEFAULT    priority 7
```

Select:

```text
ORDER BY priority ASC
LIMIT 1
```

Only active, non-deleted, currently effective values participate.

---

# 51. Effective Dates

Values may be time-bound:

```text
effective_from
effective_to
```

Valid value:

```text
effective_from <= now
AND
(effective_to IS NULL OR effective_to >= now)
```

Use UTC timestamps.

Avoid overlapping active periods for the same scope unless the business rule explicitly supports them.

---

# 52. Configuration Templates

Templates can initialize:

```text
Tenant
Company
Branch
```

Example:

```text
India Manufacturing Template
```

contains:

```text
finance.tax.rounding_method = HALF_UP
finance.invoice.auto_post = false
inventory.stock.allow_negative = false
```

Applying a template must:

1. Validate all definitions.
2. Validate all values.
3. Validate scopes.
4. Apply transactionally where possible.
5. Record change history.
6. Publish events.
7. Invalidate affected cache keys.

---

# 53. Configuration Import/Export

Recommended future APIs:

```text
POST /api/v1/configuration/import
GET  /api/v1/configuration/export
```

Export should contain:

```text
definition key
scope
value
template/version metadata
```

Secrets must be excluded or exported only through an explicit secure secret-transfer mechanism.

---

# 54. Reference Validation

For a REFERENCE:

```text
finance.default_sales_account
```

the Configuration Platform should validate that:

```text
ACCOUNT_UUID
```

exists and is accessible within the current tenant/context.

Validation can use:

```text
Resource Gateway
```

or an internal reference-validation contract.

Do not blindly accept arbitrary UUIDs.

---

# 55. Frontend Generic Architecture

Recommended:

```text
ConfigurationPage
    |
    +-- ModuleNavigation
    |
    +-- CategoryNavigation
    |
    +-- ConfigurationForm
            |
            +-- SettingRenderer
                    |
                    +-- BooleanSetting
                    +-- EnumSetting
                    +-- TextSetting
                    +-- NumberSetting
                    +-- ReferenceSetting
                    +-- MultiReferenceSetting
                    +-- SecretSetting
```

The renderer consumes metadata.

---

# 56. Frontend Must Not Contain Module-Specific Setting Logic

Avoid:

```typescript
if (setting.key === "finance.tax.rounding_method") {
   ...
}
```

Prefer:

```typescript
switch (setting.ui.component) {
    case "SELECT":
        return <SelectSetting />;
    case "ASYNC_SELECT":
        return <ReferenceSetting />;
    case "SWITCH":
        return <SwitchSetting />;
}
```

---

# 57. Backend Still Validates Everything

Even if the frontend says:

```text
component = SELECT
```

the backend must validate:

```text
data_type
option exists
option active
scope allowed
permission
dependency
policy
reference validity
```

Never trust UI metadata.

---

# 58. Module Registration Lifecycle

Recommended deployment flow:

```text
Deploy Finance version 1.0
        ↓
startup registration
        ↓
module exists?
        |
        +-- no → create
        |
        +-- yes → verify version
        ↓
register categories
        ↓
register definitions
        ↓
register options
        ↓
register resources
        ↓
complete
```

Registration failures should fail the deployment/startup if required definitions are missing or invalid.

---

# 59. Migration Strategy

Use Alembic.

Do not use:

```python
Base.metadata.create_all()
```

in production.

Migration order:

```text
configuration schema
    ↓
module
    ↓
category
    ↓
resource
    ↓
definition
    ↓
option
    ↓
value
    ↓
dependency
    ↓
policy
    ↓
template
    ↓
audit/change
```

---

# 60. Testing Strategy

## Unit tests

Test:

```text
ConfigurationKey
ConfigurationScope
ConfigurationType
Validation
Resolution
Dependency evaluation
Policy evaluation
```

## Integration tests

Test:

```text
PostgreSQL
Redis
Outbox
Resource Gateway
Encryption
```

## API tests

Test:

```text
Public APIs
Internal APIs
Authorization
Tenant isolation
Scope resolution
Concurrency
Pagination
Validation
```

## Contract tests

Each module should verify that its configuration definitions can be registered against the current Configuration Platform version.

---

# 61. Critical Security Tests

Must test:

```text
Tenant A cannot read Tenant B configuration.
Tenant A cannot write Tenant B configuration.
Company A cannot access Company B.
Branch A cannot access Branch B.
Unauthorized user cannot modify settings.
Unauthorized service cannot register definitions.
Secrets are never returned by normal endpoints.
Secrets never appear in logs.
Secrets never appear in audit history.
Stale version cannot overwrite current value.
Invalid references are rejected.
```

---

# 62. Observability

Every configuration command should have:

```text
request_id
correlation_id
tenant_id
user_id
service_name
setting_key
operation
duration
result
```

Metrics:

```text
configuration_get_total
configuration_set_total
configuration_resolution_total
configuration_cache_hit_total
configuration_cache_miss_total
configuration_resolution_latency
configuration_validation_failure_total
configuration_authorization_failure_total
configuration_event_publish_failure_total
```

Never log configuration secret values.

---

# 63. Performance Targets

Recommended initial targets:

```text
Redis hit:
< 5 ms application-side target

Configuration DB lookup:
< 30 ms typical target

Bulk resolution:
prefer one request over N requests

Cache:
high hit ratio for frequently accessed settings
```

Exact production targets should be established from load testing.

---

# 64. Anti-Patterns

Never:

```text
Store all settings in one unstructured JSON blob.
```

Never:

```text
Let every module create its own settings table.
```

Never:

```text
Let frontend call arbitrary URLs stored in database.
```

Never:

```text
Use configuration to execute arbitrary APIs.
```

Never:

```text
Trust tenant/company/branch IDs from headers.
```

Never:

```text
Return decrypted secrets from generic APIs.
```

Never:

```text
Publish events before DB commit.
```

Never:

```text
Use labels as stored enum values.
```

Never:

```text
Put business transactions inside configuration.
```

Never:

```text
Make frontend UI behavior the security boundary.
```

---

# 65. Recommended Example

Definition:

```json
{
  "key": "finance.default_sales_account",
  "module": "finance",
  "category": "general",
  "display_name": "Default Sales Account",
  "data_type": "REFERENCE",
  "allowed_scopes": [
    "TENANT",
    "COMPANY",
    "BRANCH"
  ],
  "reference": {
    "resource": "finance.account"
  },
  "ui_metadata": {
    "component": "ASYNC_SELECT",
    "group": "Accounting",
    "order": 20
  }
}
```

User opens settings:

```text
Finance
  → General
```

Frontend receives metadata:

```text
REFERENCE
ASYNC_SELECT
finance.account
```

Frontend requests account options through the resource contract.

User selects:

```text
Sales Account — 400100
```

Frontend submits:

```json
{
  "value": "ACCOUNT_UUID"
}
```

Configuration service validates:

```text
user authorization
tenant
company
branch
definition
scope
reference
```

Then:

```text
UPDATE
 ↓
CHANGE HISTORY
 ↓
OUTBOX
 ↓
COMMIT
 ↓
EVENT
 ↓
CACHE INVALIDATION
```

---

# 66. Recommended Final Database Objects

Core:

```text
configuration_module
configuration_category
configuration_definition
configuration_option
configuration_resource
configuration_value
configuration_dependency
configuration_policy
configuration_secret
configuration_template
configuration_template_value
configuration_change
configuration_audit
```

Shared platform:

```text
outbox_event
```

Optional later:

```text
configuration_import
configuration_export
configuration_release
configuration_definition_version
```

Do not add optional tables until there is a real use case.

---

# 67. Definition of Done

The Configuration Platform is production-ready when:

- [x] P28-LIVE-002: secret *values* stored as ciphertext only; public GET never returns `_secret_plain` / plaintext


### Architecture

- [ ] DDD boundaries are respected.
- [ ] Domain does not depend on infrastructure.
- [ ] CQRS is implemented.
- [ ] Configuration is independently deployable/extractable.

### Multi-tenancy

- [ ] Tenant isolation is enforced.
- [ ] Company hierarchy is validated.
- [ ] Branch hierarchy is validated.
- [ ] Context comes from authenticated token.

### Configuration

- [ ] Module registration works.
- [ ] Category registration works.
- [ ] Definition registration works.
- [ ] ENUM options work.
- [ ] Localized labels work.
- [ ] REFERENCE works.
- [ ] MULTI_REFERENCE works.
- [ ] Validation works.
- [ ] Dependencies work.
- [ ] Scope inheritance works.
- [ ] Reset/override works.
- [ ] Effective configuration works.

### Dynamic UI

- [ ] UI metadata is exposed.
- [ ] Generic controls render from metadata.
- [ ] Dynamic references use resource registry.
- [ ] Conditional visibility works.
- [ ] Frontend has no module-specific setting logic.

### Security

- [ ] Public/internal APIs are separated.
- [ ] Service-to-service authentication works.
- [ ] Secret encryption works.
- [ ] Secret access is audited.
- [ ] Secrets never appear in logs.
- [ ] Authorization is enforced server-side.

### Reliability

- [ ] Optimistic locking works.
- [ ] Transactional outbox works.
- [ ] Event consumers are idempotent.
- [ ] Redis invalidation works.
- [ ] PostgreSQL remains source of truth.

### Operations

- [ ] Metrics exist.
- [ ] Structured logs exist.
- [ ] Request/correlation IDs exist.
- [ ] Alembic migrations exist.
- [ ] Backup/restore has been tested.
- [ ] Load tests exist.
- [ ] Security tests exist.

---

# 68. Final Engineering Contract

The following rules are mandatory for all JeslotERP modules:

```text
MODULE DEFINES
      ↓
CONFIGURATION STORES
      ↓
CONFIGURATION RESOLVES
      ↓
FRONTEND RENDERS FROM METADATA
      ↓
MODULE CONSUMES THROUGH SDK/API
      ↓
CONFIGURATION CHANGE PRODUCES EVENT
      ↓
MODULE DECIDES BUSINESS REACTION
```

The Configuration Platform is therefore:

```text
NOT:
    an arbitrary API executor

NOT:
    a replacement for module master data

NOT:
    a collection of module-specific settings tables

YES:
    a centralized configuration registry

YES:
    a hierarchical configuration resolver

YES:
    a metadata-driven settings platform

YES:
    a resource/reference registry

YES:
    a validation and policy engine

YES:
    a secure secret configuration layer

YES:
    a cache-backed configuration service

YES:
    an event-producing bounded context

YES:
    a platform that can later become
    jeslot-configuration-service
```

This design keeps JeslotERP configuration centralized like an ERP platform while preserving DDD boundaries and allowing Finance, Inventory, HR, CRM, and future modules to remain independently evolvable.
