# JeslotERP Configuration Platform — Final Production Schema

**Version:** 1.3  
**Last reviewed:** 2026-09-08  
**Status:** Production Schema Baseline (merchant-hydraulic-backend)

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial production schema baseline. |
| 1.1 | 2026-09-08 | Added the missing `configuration_value` scope uniqueness (§19/§28) and scope-integrity + effective-date CHECKs (§18); added DDL for the previously prose-only `configuration_secret`, `configuration_template`, `configuration_template_value`, `configuration_change`, `configuration_audit` tables (§28.1); made all natural-key uniques soft-delete-safe partial indexes (§29); renamed the optimistic-lock column `version_no → version` and the module semantic column `version → module_version` to match the ORG platform and remove the duplicate `version` name; aligned `configuration_secret` with the full scope model; gave `configuration_option` lifecycle columns + one-default rule. |
| 1.2 | 2026-09-08 | Added outbox DDL ownership; expanded RLS GUC contract; aligned permission catalog with Complete API; fixed secret prose columns + scopes path; documented `ui_metadata`↔`ui` mapping and consumer aliases; updated folder structure to match repo ModulePlugin layout. |
| 1.3 | 2026-09-08 | Froze package `platforms.p03_configuration`. |

---

# 0. Repository Integration Contract

| Item | Frozen value |
|---|---|
| Docs folder | `docs/platforms/03_configuration/` |
| Platform package | `platforms.p03_configuration` |
| PostgreSQL schema | `configuration` |
| Outbox table | `configuration.outbox_event` |
| Idempotency table | `configuration.configuration_idempotency_key` |
| Alembic | reviewed migrations only; require PostgreSQL 15+ for `NULLS NOT DISTINCT` partial uniques |
| Consumer aliases | `ConfigurationCoreSetting` / `ConfigurationCoreSettingValue` → definition/value |

DDL in this document is authoritative for tables/columns/constraints. HTTP inventories below are summaries — Complete API wins for paths/verbs.

## 1. Purpose

The JeslotERP Configuration Platform is a centralized, metadata-driven configuration bounded context for the complete ERP ecosystem.

It provides:

- Centralized configuration definitions
- Tenant/company/branch/user/role scoped values
- Hierarchical inheritance and effective-value resolution
- Module registration
- Category registration
- Dynamic UI metadata
- Enum/static options with display labels
- Dynamic reference/API-backed options
- Resource registry for cross-module references
- Validation rules
- Conditional visibility/dependencies
- Configuration policies
- Secret configuration with encryption
- Configuration templates
- Versioning and optimistic concurrency
- Audit/change history
- Redis caching
- Transactional outbox events
- Internal module APIs
- Public administration APIs
- SDK-compatible consumption APIs

The core principle is:

> Modules own the meaning and definition of their configuration. The Configuration Platform owns storage, scope resolution, validation, inheritance, caching, audit, metadata, and configuration APIs.

---

# 2. Architectural Ownership

```text
Finance Module
    |
    | registers definitions
    v
Configuration Platform
    |
    +-- stores definitions
    +-- stores values
    +-- resolves effective values
    +-- exposes metadata
    +-- validates values
    +-- manages inheritance
    +-- manages secrets
    +-- publishes configuration events
    |
    v
Redis / Outbox / Event Bus
```

A module MUST NOT directly write to the configuration database.

A module may:

1. Register its configuration definitions through an internal API/SDK.
2. Read effective configuration through the configuration API/SDK.
3. Receive `configuration.setting.updated` events.
4. Decide its own business behavior after receiving a configuration event.

The Configuration Platform MUST NOT arbitrarily execute another module's business API because a setting changed.

---

# 3. Configuration vs Business Data

Configuration is for behavior and policy:

```text
finance.invoice.auto_post
finance.tax.rounding_method
inventory.stock.allow_negative
sales.invoice.require_approval
notification.whatsapp.enabled
```

Business/master data remains owned by the appropriate bounded context:

```text
Customer
Supplier
Item
Warehouse
Account
Tax
Employee
Invoice
Sales Order
Stock Transaction
```

A configuration reference may point to master data without owning that master data.

---

# 4. Scope Model

Supported scopes:

```text
SYSTEM
TENANT
COMPANY
BRANCH
ROLE
USER
```

Optional future scopes:

```text
DEPARTMENT
WAREHOUSE
EMPLOYEE
```

Recommended resolution order:

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
```

Not every setting must allow every scope.

Example:

```json
{
  "key": "finance.tax.rounding_method",
  "allowed_scopes": [
    "TENANT",
    "COMPANY",
    "BRANCH"
  ]
}
```

---

# 5. Important Context Rule

Configuration APIs derive security context from the authenticated token.

Do not use:

```text
X-Tenant-ID
X-Company-ID
X-Branch-ID
```

as the authoritative security mechanism.

The access token should contain the active context:

```json
{
  "sub": "USER_UUID",
  "tenant_id": "TENANT_UUID",
  "company_id": "COMPANY_UUID",
  "branch_id": "BRANCH_UUID",
  "session_id": "SESSION_UUID"
}
```

Context switching issues a new context-bound access token.

---

# 6. PostgreSQL Schema

Recommended PostgreSQL schema:

```sql
CREATE SCHEMA IF NOT EXISTS configuration;
```

PostgreSQL should have `pgcrypto` enabled for UUID generation:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

---

# 7. Shared Configuration Base

All configuration-owned tables should follow the JeslotERP enterprise base model conventions.

Common fields:

```text
id
status
created_by
created_at
updated_by
updated_at
deleted_by
deleted_at
is_deleted
version
row_version
remarks
notes
metadata
custom_fields
tags
```

Configuration-specific tables may additionally use:

```text
effective_from
effective_to
```

---

# 8. configuration_module

Represents a registered ERP/module/platform owner.

Examples:

```text
finance
inventory
sales
purchase
crm
hr
payroll
organization
notification
security
system
```

## Columns

```text
id                  UUID PK
code                VARCHAR(100) NOT NULL   -- unique via partial index (§29.1)
name                VARCHAR(200) NOT NULL
description         TEXT
module_version      VARCHAR(50)             -- module's own semantic version
service_name        VARCHAR(150)
service_version     VARCHAR(50)

status              VARCHAR(30) NOT NULL
is_system           BOOLEAN NOT NULL DEFAULT false
is_enabled          BOOLEAN NOT NULL DEFAULT true

created_by          UUID
created_at          TIMESTAMPTZ
updated_by          UUID
updated_at          TIMESTAMPTZ
deleted_by          UUID
deleted_at          TIMESTAMPTZ
is_deleted          BOOLEAN NOT NULL DEFAULT false

version             INTEGER NOT NULL DEFAULT 1   -- optimistic-lock counter
row_version         UUID NOT NULL
metadata            JSONB
tags                JSONB
```

`service_name` identifies the consuming service but does not expose internal service URLs.

---

# 9. configuration_category

Groups settings for UI and organization.

Examples:

```text
finance
    general
    invoice
    tax
    payment

inventory
    general
    stock
    warehouse
    valuation
```

## Columns

```text
id
module_id           UUID FK configuration_module.id

code                VARCHAR(100)
name                VARCHAR(200)
description         TEXT

parent_id           UUID NULL
display_order       INTEGER DEFAULT 0

status
is_hidden
is_system

created_by
created_at
updated_by
updated_at
deleted_by
deleted_at
is_deleted

version
row_version

metadata
tags
```

Unique:

```text
(module_id, code)
```

---

# 10. configuration_definition

This is the central table.

It defines WHAT a configuration setting is.

Example:

```text
finance.tax.rounding_method
```

## Columns

```text
id                  UUID PK

module_id           UUID FK
category_id         UUID FK

key                 VARCHAR(255) UNIQUE NOT NULL
display_name        VARCHAR(255) NOT NULL
description         TEXT

data_type           VARCHAR(50) NOT NULL

default_value       JSONB
validation_rules    JSONB

allowed_scopes      JSONB NOT NULL

is_required         BOOLEAN DEFAULT false
is_editable         BOOLEAN DEFAULT true
is_hidden           BOOLEAN DEFAULT false
is_secret           BOOLEAN DEFAULT false
is_system           BOOLEAN DEFAULT false

restart_required    BOOLEAN DEFAULT false

display_order       INTEGER DEFAULT 0

ui_metadata         JSONB

reference_resource_id UUID NULL

status
created_by
created_at
updated_by
updated_at
deleted_by
deleted_at
is_deleted

version
row_version

remarks
notes
metadata
custom_fields
tags
```

---

# 11. Configuration Data Types

Recommended values:

```text
STRING
TEXT
INTEGER
DECIMAL
BOOLEAN
DATE
DATETIME
TIME
ENUM
JSON
REFERENCE
MULTI_REFERENCE
LIST
MAP
SECRET
```

Example:

```text
BOOLEAN
```

renders as a switch.

```text
ENUM
```

renders as a select.

```text
REFERENCE
```

renders as an async lookup.

---

# 12. configuration_option

Do not store enum options only as:

```json
[
  "HALF_UP",
  "HALF_DOWN"
]
```

Store value and display metadata separately.

## Columns

```text
id                  UUID PK
definition_id       UUID FK

value               VARCHAR(255) NOT NULL
display_name        VARCHAR(255) NOT NULL

display_order       INTEGER DEFAULT 0

is_active           BOOLEAN DEFAULT true
is_default          BOOLEAN DEFAULT false

labels              JSONB
metadata            JSONB

created_at
updated_at
```

Example:

```json
{
  "value": "HALF_UP",
  "display_name": "Half Up",
  "labels": {
    "en": "Half Up",
    "gu": "હાફ અપ",
    "hi": "हाफ अप"
  }
}
```

The stored configuration value is:

```text
HALF_UP
```

The frontend displays:

```text
Half Up
```

---

# 13. Why value and label must be separate

Never store:

```text
Half Up
```

as the actual business value.

Store:

```text
HALF_UP
```

because labels may change or be translated.

For example:

```text
HALF_UP
```

can display as:

```text
Half Up
Հाफ अप
હાફ અપ
```

without changing existing configuration data.

---

# 14. configuration_resource

A resource registry allows configuration settings to dynamically reference data owned by another module.

Examples:

```text
finance.account
organization.warehouse
organization.branch
hr.employee
sales.price_list
tax.tax_category
payment.gateway
```

## Columns

```text
id                  UUID PK

code                VARCHAR(150) UNIQUE NOT NULL
module_id           UUID FK

name                VARCHAR(255)
description         TEXT

resource_type       VARCHAR(50)

label_field         VARCHAR(100)
value_field         VARCHAR(100)

api_contract        JSONB

search_supported    BOOLEAN DEFAULT true
pagination_supported BOOLEAN DEFAULT true

status
is_system
is_enabled

created_at
updated_at
metadata
```

Important:

`api_contract` describes the logical API contract.

Do not store private service URLs in configuration values.

Example:

```json
{
  "resource": "finance.account",
  "operations": {
    "list": "LIST",
    "search": "SEARCH",
    "get": "GET"
  }
}
```

The API Gateway/service registry resolves the actual service endpoint.

---

# 15. REFERENCE configuration

Example:

```json
{
  "key": "finance.default_sales_account",
  "data_type": "REFERENCE",
  "reference": {
    "resource": "finance.account"
  },
  "ui": {
    "component": "ASYNC_SELECT"
  }
}
```

Frontend flow:

```text
Configuration Definition
        ↓
data_type = REFERENCE
        ↓
resource = finance.account
        ↓
Resource Registry
        ↓
API Gateway
        ↓
Finance Account API
```

The frontend does not hardcode:

```text
if setting == "finance.default_sales_account":
    call finance accounts API
```

---

# 16. MULTI_REFERENCE

For settings selecting multiple records:

```text
finance.allowed_tax_accounts
```

Definition:

```json
{
  "data_type": "MULTI_REFERENCE",
  "reference": {
    "resource": "finance.account"
  },
  "ui": {
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

# 17. configuration_value

Stores the actual configured value.

## Columns

```text
id                  UUID PK

definition_id       UUID FK

scope_type          VARCHAR(30) NOT NULL

tenant_id           UUID NULL
company_id          UUID NULL
branch_id           UUID NULL
role_id             UUID NULL
user_id             UUID NULL

value               JSONB NOT NULL

effective_from      TIMESTAMPTZ NULL
effective_to        TIMESTAMPTZ NULL

status
created_by
created_at
updated_by
updated_at
deleted_by
deleted_at
is_deleted

version
row_version

remarks
notes
metadata
custom_fields
tags
```

---

# 18. Scope Integrity

Examples:

## TENANT

```text
scope_type = TENANT
tenant_id = T1
company_id = NULL
branch_id = NULL
role_id = NULL
user_id = NULL
```

## COMPANY

```text
scope_type = COMPANY
tenant_id = T1
company_id = C1
branch_id = NULL
role_id = NULL
user_id = NULL
```

## BRANCH

```text
scope_type = BRANCH
tenant_id = T1
company_id = C1
branch_id = B1
role_id = NULL
user_id = NULL
```

## USER

```text
scope_type = USER
tenant_id = T1
company_id = C1
branch_id = B1
role_id = NULL
user_id = U1
```

The application layer must validate:

```text
company belongs to tenant
branch belongs to company
user belongs to tenant
role belongs to applicable tenant/context
```

---

# 19. Unique Configuration Value

Recommended logical uniqueness:

```text
(
    definition_id,
    scope_type,
    tenant_id,
    company_id,
    branch_id,
    role_id,
    user_id,
    effective_from
)
```

PostgreSQL NULL handling must be addressed using either:

1. PostgreSQL NULLS NOT DISTINCT unique indexes, or
2. normalized scope columns, or
3. application + database constraints.

For production PostgreSQL, prefer database-enforced uniqueness. **This is implemented** as the partial unique index `uq_config_value_scope` in §29.1 (`NULLS NOT DISTINCT`, scoped to active, non-time-boxed rows). The scope columns themselves are additionally guarded by the `ck_configuration_value_scope` CHECK (§28) so only the correct columns are populated per `scope_type`.

---

# 20. configuration_dependency

Supports dependencies between settings.

Example:

```text
notification.whatsapp.enabled
```

controls:

```text
notification.whatsapp.provider
```

## Columns

```text
id
definition_id
depends_on_definition_id

condition_operator
condition_value

effect_type

metadata
status
created_at
updated_at
```

Example:

```json
{
  "definition": "notification.whatsapp.provider",
  "depends_on": "notification.whatsapp.enabled",
  "operator": "EQUALS",
  "value": true,
  "effect": "SHOW"
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

# 21. configuration_policy

Policies define additional behavior around settings.

Examples:

```text
MFA required
Setting can only be changed by tenant admin
Secret cannot be returned
Setting cannot be changed at branch level
```

## Columns

```text
id
definition_id

policy_type
policy_config JSONB

status
created_at
updated_at
created_by
updated_by

metadata
```

---

# 22. configuration_secret

Secrets should be separated logically from normal values.

Examples:

```text
smtp.password
payment.razorpay.secret
whatsapp.api_secret
gst.api_secret
```

## Columns

```text
id
definition_id

scope_type
tenant_id
company_id
branch_id
role_id
user_id

encrypted_value
encryption_key_id

algorithm
key_version

configured_at
last_rotated_at
expires_at

status
is_deleted

created_by
created_at
updated_by
updated_at

version
row_version

metadata
```

Prose columns must match DDL §28.1. `scope_type` + full scope FK set are required.

Use authenticated encryption such as AES-256-GCM or a managed KMS/secret manager.

Never return decrypted secrets from normal effective-configuration endpoints.

---

# 23. Secret API behavior

Normal API:

```http
GET /api/v1/configuration/settings/notification.smtp.password
```

returns:

```json
{
  "key": "notification.smtp.password",
  "is_secret": true,
  "configured": true,
  "value": null
}
```

Authorized internal service access must use:

```http
POST /internal/v1/configuration/secrets/{key}/resolve
```

(Never use GET for plaintext secret material.)

---

# 24. configuration_template

Allows predefined configuration packages.

Example:

```text
India Manufacturing Template
Retail Template
Service Business Template
```

## Columns

```text
id
code
name
description
template_version

module_id
status
is_system

created_by
created_at
updated_by
updated_at

metadata
tags
```

---

# 25. configuration_template_value

Stores template values.

```text
id
template_id
definition_id

value
scope_type

created_at
updated_at
metadata
```

Template application should create normal `configuration_value` records.

Templates must not bypass validation.

---

# 26. configuration_change

Stores immutable configuration change history.

```text
id

definition_id
value_id

scope_type

tenant_id
company_id
branch_id
role_id
user_id

old_value
new_value

action
reason

changed_by
changed_at

request_id
correlation_id

metadata
```

Actions:

```text
CREATE
UPDATE
RESET
DELETE
RESTORE
IMPORT
TEMPLATE_APPLY
```

Secrets must never be written as plaintext into history.

---

# 27. configuration_audit

Security/audit event stream.

Examples:

```text
SETTING_CREATED
SETTING_UPDATED
SETTING_RESET
SECRET_ACCESSED
SECRET_ROTATED
DEFINITION_CREATED
DEFINITION_UPDATED
MODULE_REGISTERED
TEMPLATE_APPLIED
```

Include:

```text
actor_user_id
tenant_id
company_id
branch_id
request_id
correlation_id
ip_address
user_agent
timestamp
metadata
```

Do not store sensitive secret values.

---

# 28. Database DDL — Core Tables

```sql
CREATE TABLE configuration.configuration_module (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(100) NOT NULL,             -- uniqueness enforced by a soft-delete-safe partial index (§29)
    name VARCHAR(200) NOT NULL,
    description TEXT,
    module_version VARCHAR(50),             -- module's own semantic version (e.g. 1.0.0)
    service_name VARCHAR(150),
    service_version VARCHAR(50),

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    is_enabled BOOLEAN NOT NULL DEFAULT TRUE,

    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,
    deleted_by UUID,
    deleted_at TIMESTAMPTZ,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    version INTEGER NOT NULL DEFAULT 1,      -- optimistic-lock counter (matches ORG platform)
    row_version UUID NOT NULL DEFAULT gen_random_uuid(),

    metadata JSONB,
    tags JSONB
);

CREATE TABLE configuration.configuration_category (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id UUID NOT NULL
        REFERENCES configuration.configuration_module(id),

    code VARCHAR(100) NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,

    parent_id UUID NULL
        REFERENCES configuration.configuration_category(id),

    display_order INTEGER NOT NULL DEFAULT 0,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,

    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,
    deleted_by UUID,
    deleted_at TIMESTAMPTZ,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    version INTEGER NOT NULL DEFAULT 1,
    row_version UUID NOT NULL DEFAULT gen_random_uuid(),

    metadata JSONB,
    tags JSONB
    -- (module_id, code) uniqueness enforced by a soft-delete-safe partial index (§29)
);

CREATE TABLE configuration.configuration_resource (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    module_id UUID NOT NULL
        REFERENCES configuration.configuration_module(id),

    code VARCHAR(150) NOT NULL,             -- unique via soft-delete-safe partial index (§29)
    name VARCHAR(255) NOT NULL,
    description TEXT,

    resource_type VARCHAR(50) NOT NULL,

    label_field VARCHAR(100),
    value_field VARCHAR(100),

    api_contract JSONB,

    search_supported BOOLEAN NOT NULL DEFAULT TRUE,
    pagination_supported BOOLEAN NOT NULL DEFAULT TRUE,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    is_enabled BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ,

    metadata JSONB
);

CREATE TABLE configuration.configuration_definition (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    module_id UUID NOT NULL
        REFERENCES configuration.configuration_module(id),

    category_id UUID
        REFERENCES configuration.configuration_category(id),

    key VARCHAR(255) NOT NULL,              -- unique via soft-delete-safe partial index (§29)

    display_name VARCHAR(255) NOT NULL,
    description TEXT,

    data_type VARCHAR(50) NOT NULL,

    default_value JSONB,
    validation_rules JSONB,

    allowed_scopes JSONB NOT NULL,

    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    is_editable BOOLEAN NOT NULL DEFAULT TRUE,
    is_hidden BOOLEAN NOT NULL DEFAULT FALSE,
    is_secret BOOLEAN NOT NULL DEFAULT FALSE,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,

    restart_required BOOLEAN NOT NULL DEFAULT FALSE,

    display_order INTEGER NOT NULL DEFAULT 0,

    ui_metadata JSONB,

    reference_resource_id UUID NULL
        REFERENCES configuration.configuration_resource(id),

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,
    deleted_by UUID,
    deleted_at TIMESTAMPTZ,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    version INTEGER NOT NULL DEFAULT 1,
    row_version UUID NOT NULL DEFAULT gen_random_uuid(),

    remarks TEXT,
    notes TEXT,
    metadata JSONB,
    custom_fields JSONB,
    tags JSONB
);

CREATE TABLE configuration.configuration_option (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    definition_id UUID NOT NULL
        REFERENCES configuration.configuration_definition(id)
        ON DELETE CASCADE,

    value VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,

    display_order INTEGER NOT NULL DEFAULT 0,

    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    is_default BOOLEAN NOT NULL DEFAULT FALSE,

    labels JSONB,
    metadata JSONB,

    -- Base-model lifecycle (§7): options are soft-deleted like every other config table.
    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,
    version INTEGER NOT NULL DEFAULT 1
    -- (definition_id, value) uniqueness and the one-default rule are enforced by
    -- soft-delete-safe partial indexes (§29)
);

CREATE TABLE configuration.configuration_value (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    definition_id UUID NOT NULL
        REFERENCES configuration.configuration_definition(id),

    scope_type VARCHAR(30) NOT NULL,

    tenant_id UUID,
    company_id UUID,
    branch_id UUID,
    role_id UUID,
    user_id UUID,

    value JSONB NOT NULL,

    effective_from TIMESTAMPTZ,
    effective_to TIMESTAMPTZ,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,
    deleted_by UUID,
    deleted_at TIMESTAMPTZ,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    version INTEGER NOT NULL DEFAULT 1,
    row_version UUID NOT NULL DEFAULT gen_random_uuid(),

    remarks TEXT,
    notes TEXT,
    metadata JSONB,
    custom_fields JSONB,
    tags JSONB,

    -- Scope integrity (§18): the right scope columns are populated for each scope_type.
    CONSTRAINT ck_configuration_value_scope CHECK (
        (scope_type = 'SYSTEM'  AND tenant_id IS NULL AND company_id IS NULL AND branch_id IS NULL AND role_id IS NULL AND user_id IS NULL) OR
        (scope_type = 'TENANT'  AND tenant_id IS NOT NULL AND company_id IS NULL AND branch_id IS NULL) OR
        (scope_type = 'COMPANY' AND tenant_id IS NOT NULL AND company_id IS NOT NULL AND branch_id IS NULL) OR
        (scope_type = 'BRANCH'  AND tenant_id IS NOT NULL AND company_id IS NOT NULL AND branch_id IS NOT NULL) OR
        (scope_type = 'ROLE'    AND tenant_id IS NOT NULL AND role_id IS NOT NULL) OR
        (scope_type = 'USER'    AND user_id IS NOT NULL)
    ),
    CONSTRAINT ck_configuration_value_effective CHECK (
        effective_to IS NULL OR effective_from IS NULL OR effective_to >= effective_from
    )
);

CREATE TABLE configuration.configuration_dependency (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    definition_id UUID NOT NULL
        REFERENCES configuration.configuration_definition(id)
        ON DELETE CASCADE,

    depends_on_definition_id UUID NOT NULL
        REFERENCES configuration.configuration_definition(id)
        ON DELETE CASCADE,

    condition_operator VARCHAR(50) NOT NULL,
    condition_value JSONB,

    effect_type VARCHAR(30) NOT NULL,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ,

    metadata JSONB
);

CREATE TABLE configuration.configuration_policy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    definition_id UUID
        REFERENCES configuration.configuration_definition(id)
        ON DELETE CASCADE,

    policy_type VARCHAR(100) NOT NULL,
    policy_config JSONB NOT NULL,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,

    metadata JSONB
);
```

> **Note on `ON DELETE CASCADE` + soft delete.** `configuration_option`, `_dependency` and `_policy` cascade from `configuration_definition`, but definitions are normally **soft-deleted** (`is_deleted = true`), so the cascade only fires on a genuine hard delete — which should be an admin-only maintenance action. Application code must soft-delete children alongside a soft-deleted definition; the FK cascade is a safety net for hard deletes only, not the normal path.

---

# 28.1. Database DDL — Secret, Template, History & Audit Tables

These tables were described in §22–§27; their production DDL follows. Secret and history/audit tables intentionally reuse the same scope model as `configuration_value`.

```sql
CREATE TABLE configuration.configuration_secret (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    definition_id UUID NOT NULL
        REFERENCES configuration.configuration_definition(id),

    -- Same scope model as configuration_value so secrets resolve identically.
    scope_type VARCHAR(30) NOT NULL,
    tenant_id UUID,
    company_id UUID,
    branch_id UUID,
    role_id UUID,
    user_id UUID,

    encrypted_value BYTEA NOT NULL,        -- ciphertext only; never plaintext
    encryption_key_id VARCHAR(150) NOT NULL,
    algorithm VARCHAR(50) NOT NULL DEFAULT 'AES-256-GCM',
    key_version INTEGER NOT NULL DEFAULT 1,

    configured_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_rotated_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,

    version INTEGER NOT NULL DEFAULT 1,
    row_version UUID NOT NULL DEFAULT gen_random_uuid(),

    metadata JSONB,

    CONSTRAINT ck_configuration_secret_scope CHECK (
        (scope_type = 'SYSTEM'  AND tenant_id IS NULL AND company_id IS NULL AND branch_id IS NULL AND role_id IS NULL AND user_id IS NULL) OR
        (scope_type = 'TENANT'  AND tenant_id IS NOT NULL AND company_id IS NULL AND branch_id IS NULL) OR
        (scope_type = 'COMPANY' AND tenant_id IS NOT NULL AND company_id IS NOT NULL AND branch_id IS NULL) OR
        (scope_type = 'BRANCH'  AND tenant_id IS NOT NULL AND company_id IS NOT NULL AND branch_id IS NOT NULL) OR
        (scope_type = 'ROLE'    AND tenant_id IS NOT NULL AND role_id IS NOT NULL) OR
        (scope_type = 'USER'    AND user_id IS NOT NULL)
    )
);

CREATE TABLE configuration.configuration_template (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    code VARCHAR(100) NOT NULL,            -- unique via partial index (§29)
    name VARCHAR(200) NOT NULL,
    description TEXT,
    template_version VARCHAR(50),

    module_id UUID
        REFERENCES configuration.configuration_module(id),

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by UUID,
    updated_at TIMESTAMPTZ,

    version INTEGER NOT NULL DEFAULT 1,
    row_version UUID NOT NULL DEFAULT gen_random_uuid(),

    metadata JSONB,
    tags JSONB
);

CREATE TABLE configuration.configuration_template_value (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    template_id UUID NOT NULL
        REFERENCES configuration.configuration_template(id)
        ON DELETE CASCADE,

    definition_id UUID NOT NULL
        REFERENCES configuration.configuration_definition(id),

    value JSONB NOT NULL,
    scope_type VARCHAR(30) NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ,
    metadata JSONB,

    CONSTRAINT uq_configuration_template_value
        UNIQUE (template_id, definition_id, scope_type)
);

-- Immutable change history: append-only, never updated in place.
CREATE TABLE configuration.configuration_change (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    definition_id UUID NOT NULL
        REFERENCES configuration.configuration_definition(id),
    value_id UUID,   -- no FK: history must survive deletion of the value row

    scope_type VARCHAR(30) NOT NULL,
    tenant_id UUID,
    company_id UUID,
    branch_id UUID,
    role_id UUID,
    user_id UUID,

    old_value JSONB,   -- redacted for secret definitions
    new_value JSONB,   -- redacted for secret definitions

    action VARCHAR(30) NOT NULL,   -- CREATE | UPDATE | RESET | DELETE | RESTORE | IMPORT | TEMPLATE_APPLY

    reason TEXT,
    changed_by UUID,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    request_id UUID,
    correlation_id UUID,

    metadata JSONB
);

-- Security/audit event stream.
CREATE TABLE configuration.configuration_audit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    event_type VARCHAR(60) NOT NULL,   -- SETTING_UPDATED | SECRET_ACCESSED | ...

    actor_user_id UUID,
    tenant_id UUID,
    company_id UUID,
    branch_id UUID,

    request_id UUID,
    correlation_id UUID,
    ip_address INET,
    user_agent TEXT,

    occurred_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB   -- identifiers only; never secret values
);
```

---

# 29. Recommended Indexes

```sql
CREATE INDEX ix_config_definition_module
ON configuration.configuration_definition(module_id)
WHERE is_deleted = FALSE;

CREATE INDEX ix_config_definition_category
ON configuration.configuration_definition(category_id)
WHERE is_deleted = FALSE;

CREATE INDEX ix_config_value_definition
ON configuration.configuration_value(definition_id)
WHERE is_deleted = FALSE
  AND status = 'ACTIVE';

CREATE INDEX ix_config_value_resolution
ON configuration.configuration_value (
    definition_id,
    tenant_id,
    company_id,
    branch_id,
    role_id,
    user_id
)
WHERE is_deleted = FALSE
  AND status = 'ACTIVE';

CREATE INDEX ix_config_value_effective
ON configuration.configuration_value (
    definition_id,
    effective_from,
    effective_to
)
WHERE is_deleted = FALSE
  AND status = 'ACTIVE';

CREATE INDEX ix_config_resource_module
ON configuration.configuration_resource(module_id);
```

## 29.1 Uniqueness (soft-delete-safe)

Natural/business keys use partial unique indexes scoped to live rows, so a soft-deleted row releases its key and re-registration (§45) stays idempotent:

```sql
CREATE UNIQUE INDEX uq_config_module_code
ON configuration.configuration_module (code)
WHERE is_deleted = FALSE;

CREATE UNIQUE INDEX uq_config_category_module_code
ON configuration.configuration_category (module_id, code)
WHERE is_deleted = FALSE;

CREATE UNIQUE INDEX uq_config_resource_code
ON configuration.configuration_resource (code)
WHERE is_deleted = FALSE;

CREATE UNIQUE INDEX uq_config_definition_key
ON configuration.configuration_definition (key)
WHERE is_deleted = FALSE;

CREATE UNIQUE INDEX uq_config_option_value
ON configuration.configuration_option (definition_id, value)
WHERE is_deleted = FALSE;

-- At most one default option per definition.
CREATE UNIQUE INDEX uq_config_option_default
ON configuration.configuration_option (definition_id)
WHERE is_default = TRUE AND is_deleted = FALSE;

CREATE UNIQUE INDEX uq_config_template_code
ON configuration.configuration_template (code)
WHERE is_deleted = FALSE;
```

The core `configuration_value` uniqueness from §19 — one active value per definition + scope (PG15+ `NULLS NOT DISTINCT` treats the null scope columns as equal). Time-boxed values (`effective_to` set) are excluded and governed by application/temporal logic instead:

```sql
CREATE UNIQUE INDEX uq_config_value_scope
ON configuration.configuration_value
   (definition_id, scope_type, tenant_id, company_id, branch_id, role_id, user_id)
NULLS NOT DISTINCT
WHERE is_deleted = FALSE
  AND status = 'ACTIVE'
  AND effective_to IS NULL;

-- Same one-active-secret-per-scope rule for secrets.
CREATE UNIQUE INDEX uq_config_secret_scope
ON configuration.configuration_secret
   (definition_id, scope_type, tenant_id, company_id, branch_id, role_id, user_id)
NULLS NOT DISTINCT
WHERE is_deleted = FALSE
  AND status = 'ACTIVE';
```

> On PostgreSQL 14 or older, `NULLS NOT DISTINCT` is unavailable — use `COALESCE`d sentinel scope columns (or a normalized generated key column) so the null scope slots compare as equal.

---

# 30. Dynamic UI Metadata

A definition can contain:

```json
{
  "ui": {
    "component": "SELECT",
    "order": 10,
    "group": "Tax",
    "help_text": "Determines how tax amounts are rounded"
  }
}
```

Boolean:

```json
{
  "ui": {
    "component": "SWITCH"
  }
}
```

String:

```json
{
  "ui": {
    "component": "TEXT"
  }
}
```

Integer:

```json
{
  "ui": {
    "component": "NUMBER"
  }
}
```

Reference:

```json
{
  "ui": {
    "component": "ASYNC_SELECT"
  }
}
```

---

# 31. Example ENUM Definition

```json
{
  "key": "finance.tax.rounding_method",
  "module": "finance",
  "category": "tax",

  "display_name": "Tax Rounding Method",

  "data_type": "ENUM",

  "default_value": "HALF_UP",

  "allowed_scopes": [
    "TENANT",
    "COMPANY",
    "BRANCH"
  ],

  "ui_metadata": {
    "component": "SELECT",
    "order": 10,
    "group": "Tax"
  }
}
```

Options:

```json
[
  {
    "value": "HALF_UP",
    "display_name": "Half Up",
    "labels": {
      "en": "Half Up",
      "gu": "હાફ અપ",
      "hi": "हाफ अप"
    }
  },
  {
    "value": "HALF_DOWN",
    "display_name": "Half Down",
    "labels": {
      "en": "Half Down",
      "gu": "હાફ ડાઉન",
      "hi": "हाफ डाउन"
    }
  },
  {
    "value": "UP",
    "display_name": "Round Up"
  },
  {
    "value": "DOWN",
    "display_name": "Round Down"
  }
]
```

---

# 32. Example REFERENCE Definition

```json
{
  "key": "finance.default_sales_account",

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
    "component": "ASYNC_SELECT"
  }
}
```

---

# 33. Example Boolean Definition

```json
{
  "key": "finance.invoice.auto_post",

  "data_type": "BOOLEAN",

  "default_value": false,

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

# 34. Example Conditional Configuration

```text
notification.whatsapp.enabled
```

When true:

```text
notification.whatsapp.provider
notification.whatsapp.sender
notification.whatsapp.template
```

become visible.

This is represented through `configuration_dependency`.

The frontend should evaluate metadata and current values rather than hardcoding module-specific rules.

---

# 35. API Architecture

Public API:

```text
/api/v1/configuration
```

Internal API:

```text
/internal/v1/configuration
```

---

# 36. Public APIs

## Modules

```http
GET /api/v1/configuration/modules
GET /api/v1/configuration/modules/{module_code}
```

## Categories

```http
GET /api/v1/configuration/categories
GET /api/v1/configuration/categories/{category_code}
```

## Definitions

```http
GET /api/v1/configuration/definitions
GET /api/v1/configuration/definitions/{key}
```

## Settings

```http
GET    /api/v1/configuration/settings/{key}
PUT    /api/v1/configuration/settings/{key}
DELETE /api/v1/configuration/settings/{key}
POST   /api/v1/configuration/settings/{key}/reset
GET    /api/v1/configuration/settings/{key}/history
```

## Effective configuration

```http
GET  /api/v1/configuration/effective
POST /api/v1/configuration/effective/bulk
```

## Scope

```http
GET /api/v1/configuration/scopes
```

(There is no `/scope` singular path.)

## Resources

```http
GET /api/v1/configuration/resources
GET /api/v1/configuration/resources/{resource_code}
```

---

# 37. Internal APIs

Used by modules/services.

## Register module

```http
POST /internal/v1/configuration/modules/register
```

Example:

```json
{
  "code": "finance",
  "name": "Finance",
  "module_version": "1.0.0",
  "service_name": "finance-service"
}
```

## Register category

```http
POST /internal/v1/configuration/categories/register
```

## Register definition

```http
POST /internal/v1/configuration/definitions/register
```

Example:

```json
{
  "module": "finance",
  "category": "tax",
  "key": "finance.tax.rounding_method",
  "display_name": "Tax Rounding Method",
  "data_type": "ENUM",
  "default_value": "HALF_UP",
  "allowed_scopes": [
    "TENANT",
    "COMPANY",
    "BRANCH"
  ]
}
```

## Register options

```http
POST /internal/v1/configuration/definitions/{key}/options
```

## Register resource

```http
POST /internal/v1/configuration/resources/register
```

---

# 38. Internal Resolution API

Single setting:

```http
GET /internal/v1/configuration/resolve/{key}
```

Bulk:

```http
POST /internal/v1/configuration/resolve/bulk
```

Example:

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

# 39. Effective Value Response

For UI/API transparency:

```json
{
  "key": "finance.invoice.auto_post",
  "value": true,

  "source": {
    "scope": "COMPANY",
    "tenant_id": "TENANT_UUID",
    "company_id": "COMPANY_UUID"
  },

  "inherited": true,

  "definition": {
    "data_type": "BOOLEAN",
    "display_name": "Auto Post Invoice"
  }
}
```

---

# 40. Configuration UI API

For dynamic Settings screens:

```http
GET /api/v1/configuration/ui
```

or:

```http
GET /api/v1/configuration/ui/modules/finance
```

Return:

```json
{
  "module": "finance",
  "categories": [
    {
      "code": "invoice",
      "name": "Invoice",
      "settings": [
        {
          "key": "finance.invoice.auto_post",
          "data_type": "BOOLEAN",
          "ui": {
            "component": "SWITCH"
          },
          "value": true
        }
      ]
    }
  ]
}
```

React can construct the Settings page dynamically.

---

# 41. API-Backed Dynamic Options

For:

```text
finance.default_sales_account
```

the definition only says:

```json
{
  "data_type": "REFERENCE",
  "reference": {
    "resource": "finance.account"
  }
}
```

The frontend resolves:

```text
finance.account
        ↓
resource registry
        ↓
API Gateway
        ↓
finance account endpoint
```

This prevents every setting from hardcoding API URLs.

---

# 42. Module API Behavior

A module should never do:

```text
UPDATE configuration.configuration_value
```

Instead:

```text
Finance
    ↓
Configuration SDK
    ↓
Internal Configuration API
```

or:

```text
Admin UI
    ↓
Public Configuration API
```

---

# 43. Configuration SDK

Recommended Python SDK interface:

```python
value = await config.get(
    "finance.invoice.auto_post"
)
```

Bulk:

```python
values = await config.get_many([
    "finance.invoice.auto_post",
    "finance.invoice.require_approval",
    "finance.tax.rounding_method",
])
```

Set:

```python
await config.set(
    key="finance.invoice.auto_post",
    value=True,
)
```

Register:

```python
await config.register(
    module="finance",
    definitions=FinanceConfiguration.DEFINITIONS,
)
```

---

# 44. Module Configuration Registry

Each module should define its configuration metadata in code.

Example:

```python
FINANCE_CONFIGURATION = [
    {
        "key": "finance.invoice.auto_post",
        "category": "invoice",
        "data_type": "BOOLEAN",
        "default_value": False,
        "allowed_scopes": [
            "TENANT",
            "COMPANY",
            "BRANCH"
        ]
    },
    {
        "key": "finance.tax.rounding_method",
        "category": "tax",
        "data_type": "ENUM",
        "default_value": "HALF_UP",
        "allowed_scopes": [
            "TENANT",
            "COMPANY",
            "BRANCH"
        ]
    }
]
```

On deployment/startup:

```text
Finance starts
    ↓
Configuration registry
    ↓
register definitions
    ↓
Configuration Platform
```

---

# 45. Idempotent Registration

Registration must be idempotent.

Calling:

```http
POST /internal/v1/configuration/definitions/register
```

multiple times must not create duplicate definitions.

Use:

```text
key
module
definition version
```

to determine whether the definition already exists.

---

# 46. Definition Versioning

If Finance changes:

```text
finance.tax.rounding_method
```

from version:

```text
1.0
```

to:

```text
2.0
```

the Configuration Platform should preserve existing values where compatible.

Breaking changes require explicit migration.

Never silently change the meaning of an existing configuration key.

---

# 47. Resolution Algorithm

Given:

```text
key
user_id
tenant_id
company_id
branch_id
role_id
current_time
```

resolve:

```text
1. USER
2. ROLE
3. BRANCH
4. COMPANY
5. TENANT
6. SYSTEM
7. DEFINITION DEFAULT
```

Only scopes allowed by the definition participate.

Example:

```text
USER value exists?
    ↓ no
ROLE value exists?
    ↓ no
BRANCH value exists?
    ↓ no
COMPANY value exists?
    ↓ yes
return COMPANY value
```

---

# 48. Effective Configuration Cache

Redis key strategy:

```text
config:v1:{tenant_id}:{company_id}:{branch_id}:{user_id}:{key}
```

For bulk effective configuration:

```text
config-effective:v1:{tenant_id}:{company_id}:{branch_id}:{user_id}
```

Cache should never become the source of truth.

PostgreSQL remains authoritative.

---

# 49. Cache Invalidation

Configuration update:

```text
DB transaction
    ↓
configuration_value UPDATE
    ↓
configuration_change INSERT
    ↓
outbox_event INSERT
    ↓
COMMIT
    ↓
Celery/Event Worker
    ↓
Redis invalidation
```

Event:

```json
{
  "event_type": "configuration.setting.updated",
  "setting_key": "finance.invoice.auto_post",
  "tenant_id": "T1",
  "company_id": "C1",
  "branch_id": "B1"
}
```

---

# 50. Module Reaction to Configuration Events

Finance subscribes:

```text
configuration.setting.updated
```

Finance decides:

```python
if event.setting_key == "finance.invoice.auto_post":
    await refresh_invoice_configuration()
```

The Configuration Platform does not invoke Finance's business APIs directly.

---

# 51. Transaction Rule

Setting mutation must follow:

```text
BEGIN
    UPDATE configuration_value
    INSERT configuration_change
    INSERT outbox_event
COMMIT
```

Never:

```text
UPDATE DB
COMMIT
publish event
```

because that can produce inconsistent state.

---

# 52. Security

Every configuration operation must validate:

```text
authenticated user
tenant membership
company assignment
branch assignment
role/permission
setting allowed scope
definition editability
policy
```

Example permission names (canonical list = Complete API §5):

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

# 53. Secret Security

Never:

```text
log secret
cache decrypted secret in normal Redis
return secret through UI API
store secret in audit history
store raw API key
store raw password
```

Use:

```text
KMS / Secret Manager
+
encrypted database value
+
strict internal access
+
audit
```

---

# 54. PostgreSQL Row-Level Security

Tenant-owned rows in `configuration_value`, `configuration_secret`, `configuration_change`, and `configuration_audit` MUST enable RLS.

Session GUCs (set by HTTP auth dependency, mirroring ORG):

```text
app.tenant_id
app.company_id
app.branch_id
app.user_id
app.rls_bypass
```

Example policy shape:

```sql
ALTER TABLE configuration.configuration_value ENABLE ROW LEVEL SECURITY;
CREATE POLICY configuration_value_tenant_isolation
ON configuration.configuration_value
USING (
  current_setting('app.rls_bypass', true) = 'on'
  OR tenant_id IS NULL
  OR tenant_id::text = current_setting('app.tenant_id', true)
);
```

Application authorization is still required.

RLS is defense-in-depth, not a replacement for domain authorization.

Also create `configuration.outbox_event` and `configuration.configuration_idempotency_key` in the first schema migration (same columns as ORG outbox/idempotency).

---

# 55. Multi-Tenant Isolation

Every tenant-scoped value must satisfy:

```text
tenant_id = authenticated_context.tenant_id
```

Company:

```text
company.tenant_id = context.tenant_id
```

Branch:

```text
branch.company_id = context.company_id
```

User:

```text
user belongs to tenant
```

Never trust IDs supplied by the browser.

---

# 56. Settings That Should NOT Be Editable Through Generic CRUD

Do not expose direct CRUD for:

```text
configuration_change
configuration_audit
secret storage
internal event state
cache state
```

These are controlled by commands/services.

---

# 57. Example Complete Flow

Admin changes:

```text
Finance
 → Tax
 → Rounding Method
 → Round Down
```

Frontend:

```http
PUT /api/v1/configuration/settings/finance.tax.rounding_method
```

Payload:

```json
{
  "value": "DOWN"
}
```

Backend:

```text
Authenticate
    ↓
Resolve context
    ↓
Authorize
    ↓
Load definition
    ↓
Validate data type
    ↓
Validate ENUM option
    ↓
Validate allowed scope
    ↓
Validate policy
    ↓
UPDATE configuration_value
    ↓
INSERT configuration_change
    ↓
INSERT outbox_event
    ↓
COMMIT
```

Worker:

```text
outbox
 ↓
publish configuration.setting.updated
 ↓
invalidate Redis
```

Finance:

```text
receives event
 ↓
refreshes local/in-memory configuration if required
```

---

# 58. Recommended Folder Structure

```text
platforms/
└── p03_configuration/
    ├── module.py
    ├── domain/
    ├── application/
    ├── infrastructure/
    │   ├── persistence/models/   # schema="configuration"
    │   ├── messaging/outbox/
    │   └── http/
    │       ├── api_v1.py
    │       ├── admin_api.py
    │       ├── tenant_api.py
    │       └── routers/
    └── tests/
```

See Developer Guide §0 / §8 for the full ModulePlugin contract.

---

# 59. Frontend Architecture

The frontend should have generic components:

```text
BooleanSetting
EnumSetting
TextSetting
NumberSetting
DateSetting
JsonSetting
ReferenceSetting
MultiReferenceSetting
SecretSetting
```

The UI component is selected from:

```text
definition.data_type
+
definition.ui_metadata
```

It should not contain:

```text
if finance.tax.rounding_method ...
if inventory.default_warehouse ...
```

---

# 60. Final Data Flow

```text
                 MODULE
                   |
                   | Register definition
                   v
        ┌──────────────────────┐
        │ Configuration        │
        │ Definition            │
        └──────────┬───────────┘
                   |
          ┌────────┴────────┐
          |                 |
          v                 v
       Options          Resources
          |                 |
          |                 |
          └────────┬────────┘
                   |
                   v
        ┌──────────────────────┐
        │ Configuration Value  │
        └──────────┬───────────┘
                   |
                   v
             Resolution
                   |
          ┌────────┴────────┐
          v                 v
        Redis           PostgreSQL
          |
          v
    Effective Value
          |
          v
     ERP Module / UI
```

---

# 61. Final Principles

1. Configuration Platform is a bounded context.
2. Modules own configuration definitions.
3. Configuration Platform owns configuration values.
4. Modules never directly access configuration tables.
5. Public APIs are for administration/UI.
6. Internal APIs are for module registration and service consumption.
7. Use an SDK for service-to-configuration communication.
8. Use metadata-driven UI rendering.
9. Use `value + display_name` for ENUM options.
10. Use resource registry for dynamic API-backed references.
11. Do not store private service URLs in settings.
12. Use API Gateway/service discovery for dynamic module APIs.
13. Use hierarchical scope resolution.
14. Use context from the authenticated token.
15. Do not trust tenant/company/branch headers.
16. Use Redis for fast resolution.
17. PostgreSQL remains source of truth.
18. Use transactional outbox for configuration events.
19. Consumers decide their own business reaction to configuration events.
20. Keep secrets separate and encrypted.
21. Never expose secrets through generic configuration APIs.
22. Keep immutable configuration history.
23. Validate every value against its definition.
24. Make module registration idempotent.
25. Support configuration templates.
26. Support dependencies and conditional UI.
27. Use optimistic concurrency.
28. Enforce tenant isolation.
29. Keep domain logic framework-independent.
30. Design the platform so it can later be extracted from the JeslotERP modular monolith into a standalone Configuration Service.

---

# 62. Recommended Final Module Name

```text
platforms.p03_configuration
```

Docs catalog chapter:

```text
docs/platforms/03_configuration
```

Bounded context / PostgreSQL schema:

```text
configuration
```

Service (extractable later):

```text
jeslot-configuration-service
```

Public API:

```text
/api/v1/configuration
```

Internal API:

```text
/internal/v1/configuration
```

In-process SDK:

```text
platforms.p03_configuration.application.sdk
```

This architecture gives JeslotERP a centralized, metadata-driven configurable platform while remaining compatible with DDD, CQRS, multi-tenant, token-context, modular-monolith, and future microservice extraction.
