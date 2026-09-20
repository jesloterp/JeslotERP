# [ARCHIVED encyclopedia] P26 Licensing Platform — Production-Grade Python Schema

> **HYG-015:** Navigable ship inventory is [`V2_LICENSING_SCHEMA.md`](V2_LICENSING_SCHEMA.md). This file remains the line-by-line ORM encyclopedia (~94 `license_*` tables). It does **not** authorize v1 encyclopedia REST.

# P26 Licensing Platform — Production-Grade Python Schema

**Version:** 1.3  
**Last reviewed:** 2026-09-08  
**Package:** `platforms.p26_licensing`  
**PostgreSQL schema:** `licensing`

### Revision history

| Version | Date | Changes |
|---|---|---|
| 1.0 | — | Initial licensing schema baseline. |
| 1.1 | 2026-09-08 | Production-grade foundation: base models given `created_by`/`updated_by`, `created_at` server default, `updated_at` onupdate, and integer-`version` optimistic locking wired via `version_id_col` (matches ORG/Configuration); added `GlobalEntity` so catalog tables carry timestamps/version without `tenant_id`; fixed the `uuid.UUID` vs SQLAlchemy-`UUID` name collision; added §64.1 normative Production-Grade Conventions (money `Numeric` precision, intra-platform FK policy, soft-delete via `status`, partial-unique and idempotency rules); removed the duplicate `row_version` on `license_usage`; added six developer-guide-aligned tables in §63.1 (`contract_entitlement`, `subscription_pause`, `cancellation`, `usage_reset`, `discount_rule`, `quote_conversion`) and documented `entitlement_source` as folded onto `license_entitlement_feature`; **fully expanded the financially load-bearing core tables** to the §64.1 conventions inline — real intra-platform FKs, `Numeric(18,6)` money, `mapped_column`, partial-unique/CHECK constraints — on `license_price`, `license_price_tier`, `license_subscription`, `license_subscription_item`, `license_entitlement_feature`, `license_entitlement_limit`, `license_usage`, `license_invoice`, `license_invoice_line`. |
| 1.2 | 2026-09-08 | **Completed the inline expansion across the entire schema** — every table now uses `GlobalEntity`/`TenantScopedEntity`, `PGUUID`, real intra-platform `ForeignKey`s, `Numeric(18,6)` money/quantities, `String(n)` lengths, and partial-unique (`WHERE status <> 'DELETED'`) / CHECK / dedup constraints. Added a `Base.type_annotation_map` (`dict`/`list`→JSONB, `Decimal`→`Numeric(18,6)`). The full model set (94 tables, 155 FKs) was validated end-to-end: SQLAlchemy `configure_mappers()` succeeds and every table compiles to PostgreSQL DDL. Outbox/inbox/audit intentionally remain lightweight `Base` infra tables. |
| 1.3 | 2026-09-08 | Platform package frozen as `platforms.p26_licensing`; PostgreSQL schema name `licensing` (no platform number in schema). Cross-platform refs: `p01_identity`, `p02_organization`; future platforms referenced without numbers (`payment`, `tax`, `accounting`). See `docs/PLATFORM_REGISTRY.md`. |

---

# 0. Repository Integration Contract

| Item | Frozen value |
|---|---|
| Package | `platforms.p26_licensing` |
| PostgreSQL schema | `licensing` |
| Table prefix | `license_` |
| Numbering authority | [`docs/PLATFORM_REGISTRY.md`](../../PLATFORM_REGISTRY.md) |

ORM `__table_args__` for every licensing table MUST include `{"schema": "licensing"}` (or equivalent). Do **not** use schema names like `p04` or `p23_licensing`.

External platform FKs are **not** allowed across schemas — store UUID references only and validate via gateways.

---

## 1. Scope

`p26_licensing` is a dedicated, pure Subscription & Licensing Platform.

It owns:

- Product catalog
- Modules and features
- Plans and immutable plan versions
- Prices and pricing models
- Add-ons
- Subscriptions and subscription items
- Trials
- Contracts and subscription schedules
- Entitlements and entitlement compilation
- Runtime license tokens
- On-premise licenses and activations
- Metering, usage, quotas and rate limits
- Usage rating
- Coupons and discounts
- Checkout and quotes
- Marketplace applications
- Subscription billing coordination
- Billing periods and invoice calculation data
- Payment coordination/status references
- Refund/chargeback references
- Customer billing profile required for subscription commerce
- Dunning state relevant to subscription lifecycle
- Webhooks
- Domain events
- Outbox/inbox
- Idempotency
- Notifications
- Immutable audit

## Explicitly OUT OF SCOPE

The following are separate platforms/services:

- General Ledger
- Chart of Accounts
- Journal Entries
- Accounts Receivable accounting ledger
- Accounts Payable
- Revenue Recognition
- Deferred Revenue accounting
- Bank Reconciliation
- Financial Statements
- Fixed Assets
- Cost Accounting
- Accounting Dimensions
- Payment-provider processing implementation
- Full tax engine

`p26_licensing` may store external references and commercial calculation results needed for integration, but the source of truth remains the dedicated platform.

---

# 2. Recommended Python Stack

```text
Python 3.12+
FastAPI
SQLAlchemy 2.x
Alembic
PostgreSQL 16+
Pydantic 2.x
Redis
Celery / Dramatiq / Arq
```

Recommended architectural style:

```text
Domain-Driven Design
Hexagonal Architecture
CQRS where useful
Transactional Outbox
Inbox / Idempotency
Event-driven workers
Immutable financial/commercial snapshots
```

---

# 3. Package Structure

Align with assigned platforms (`p01`/`p02`/`p03`). Do not invent numbers inside nested modules.

```text
platforms/
└── p26_licensing/
    ├── module.py
    ├── domain/
    ├── application/
    ├── infrastructure/
    │   ├── persistence/
    │   │   └── models/          # schema="licensing"
    │   ├── messaging/
    │   └── http/
    │       ├── routes.py        # SuperAdmin consumer
    │       ├── tenant_routes.py # Tenant consumer
    │       └── routers/
    └── tests/
```

Internal feature folders (catalog, subscriptions, entitlement, …) live under `domain/` / `application/` / `infrastructure/` — **without** `pNN_` prefixes.

---

# 4. Common Base Model

Use a common model for tenant-scoped mutable entities. Every concrete table must land in PostgreSQL schema `licensing`.

```python
import uuid
from datetime import datetime
from decimal import Decimal

from sqlalchemy import (
    Boolean, CheckConstraint, DateTime, ForeignKey, Index, Integer, Numeric,
    String, Text, UniqueConstraint, func, text,
)
from sqlalchemy.dialects.postgresql import INET, JSONB, UUID as PGUUID
from sqlalchemy.orm import DeclarativeBase, Mapped, declared_attr, mapped_column

LICENSING_SCHEMA = "licensing"


class Base(DeclarativeBase):
    # Platform-wide type map: bare `Mapped[dict]` / `Mapped[list]` resolve to JSONB,
    # and `Mapped[Decimal]` defaults to NUMERIC(18,6) unless a column overrides it
    # (money/quantity columns still declare Numeric explicitly for clarity).
    type_annotation_map = {
        dict: JSONB,
        list: JSONB,
        Decimal: Numeric(18, 6),
    }


class BaseEntity(Base):
    """Root for every persisted entity: identity, timestamps, audit actors,
    optimistic-lock version, and a JSONB metadata bag."""
    __abstract__ = True

    id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
        server_default=func.gen_random_uuid(),
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False,
        server_default=func.now(),
    )
    created_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    updated_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        onupdate=func.now(),
    )
    updated_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    # Optimistic concurrency: SQLAlchemy auto-increments this on UPDATE and
    # raises StaleDataError on a concurrent write. Same integer-`version`
    # convention as the ORG and Configuration platforms (carried in If-Match).
    version: Mapped[int] = mapped_column(Integer, nullable=False, default=1)

    metadata_: Mapped[dict] = mapped_column(
        "metadata", JSONB, nullable=False, default=dict, server_default="{}"
    )

    @declared_attr
    def __mapper_args__(cls):
        return {"version_id_col": cls.version}


class GlobalEntity(BaseEntity):
    """Global/catalog entities (product, plan, feature, price, meter, coupon,
    signing key, …). Same base columns as BaseEntity but deliberately NO
    tenant_id — the catalog is shared across tenants."""
    __abstract__ = True


class TenantScopedEntity(BaseEntity):
    """Tenant-owned entities (subscriptions, usage, invoices, …)."""
    __abstract__ = True

    tenant_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), nullable=False, index=True,
    )
    company_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), index=True,
    )
    branch_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), index=True,
    )
```

**Base-class rule (normative):** every model extends exactly one of these — `GlobalEntity` for shared catalog rows, `TenantScopedEntity` for tenant-owned rows. No model should extend the raw `DeclarativeBase` (`Base`) and hand-declare only `id`, because that silently drops timestamps, audit actors, version and metadata. Where a snippet below still shows `class LicenseX(Base)` with a manual `id`, read it as `class LicenseX(GlobalEntity)` with the `id` line removed — see §64.1.

`tenant_id`, `company_id`, `branch_id` and any user/customer reference are cross-context identifiers (owned by `p02_organization` / `p01_identity`), so they carry **no** foreign key. Intra-licensing references **do** — see §64.1.

**Schema rule (normative):** every concrete `__table_args__` MUST include `{"schema": LICENSING_SCHEMA}` (`licensing`). Never use a numbered schema name.

---

# 5. Enumerations

```python
from enum import StrEnum


class Status(StrEnum):
    ACTIVE = "ACTIVE"
    INACTIVE = "INACTIVE"
    SUSPENDED = "SUSPENDED"
    DELETED = "DELETED"


class ProductType(StrEnum):
    PLATFORM = "PLATFORM"
    MODULE = "MODULE"
    ADDON = "ADDON"
    APP = "APP"
    SERVICE = "SERVICE"
    API = "API"
    STORAGE = "STORAGE"
    SUPPORT = "SUPPORT"


class PlanType(StrEnum):
    FREE = "FREE"
    STARTER = "STARTER"
    PRO = "PRO"
    BUSINESS = "BUSINESS"
    ENTERPRISE = "ENTERPRISE"
    CUSTOM = "CUSTOM"


class PricingModel(StrEnum):
    FREE = "FREE"
    FLAT = "FLAT"
    PER_UNIT = "PER_UNIT"
    TIERED = "TIERED"
    VOLUME = "VOLUME"
    GRADUATED = "GRADUATED"
    PACKAGE = "PACKAGE"
    STAIR_STEP = "STAIR_STEP"
    METERED = "METERED"
    HYBRID = "HYBRID"


class BillingInterval(StrEnum):
    DAY = "DAY"
    WEEK = "WEEK"
    MONTH = "MONTH"
    QUARTER = "QUARTER"
    YEAR = "YEAR"
    CUSTOM = "CUSTOM"


class SubscriptionStatus(StrEnum):
    DRAFT = "DRAFT"
    TRIALING = "TRIALING"
    ACTIVE = "ACTIVE"
    PAST_DUE = "PAST_DUE"
    PAUSED = "PAUSED"
    SUSPENDED = "SUSPENDED"
    CANCELED = "CANCELED"
    EXPIRED = "EXPIRED"


class SubscriptionChangeType(StrEnum):
    UPGRADE = "UPGRADE"
    DOWNGRADE = "DOWNGRADE"
    QUANTITY_INCREASE = "QUANTITY_INCREASE"
    QUANTITY_DECREASE = "QUANTITY_DECREASE"
    ADDON_ADD = "ADDON_ADD"
    ADDON_REMOVE = "ADDON_REMOVE"
    PRICE_CHANGE = "PRICE_CHANGE"
    CURRENCY_CHANGE = "CURRENCY_CHANGE"
    INTERVAL_CHANGE = "INTERVAL_CHANGE"
    PAUSE = "PAUSE"
    RESUME = "RESUME"
    CANCEL = "CANCEL"
    REACTIVATE = "REACTIVATE"


class EffectiveType(StrEnum):
    IMMEDIATE = "IMMEDIATE"
    END_OF_TERM = "END_OF_TERM"
    SCHEDULED = "SCHEDULED"


class FeatureType(StrEnum):
    BOOLEAN = "BOOLEAN"
    QUOTA = "QUOTA"
    LIMIT = "LIMIT"
    RATE_LIMIT = "RATE_LIMIT"
    METER = "METER"
    ACTION = "ACTION"
    MODULE = "MODULE"
    API = "API"


class AggregationType(StrEnum):
    COUNT = "COUNT"
    SUM = "SUM"
    MAX = "MAX"
    MIN = "MIN"
    UNIQUE = "UNIQUE"
    LAST = "LAST"
    AVERAGE = "AVERAGE"


class LimitType(StrEnum):
    MAX = "MAX"
    MIN = "MIN"
    RATE = "RATE"
    QUOTA = "QUOTA"
    CONCURRENT = "CONCURRENT"
    TOTAL = "TOTAL"


class EntryType(StrEnum):
    CONSUME = "CONSUME"
    RESERVE = "RESERVE"
    RELEASE = "RELEASE"
    ADJUSTMENT = "ADJUSTMENT"
    RESET = "RESET"
    REVERSAL = "REVERSAL"
```

---

# 6. Currency

```python
class LicenseCurrency(GlobalEntity):
    __tablename__ = "license_currency"

    code: Mapped[str] = mapped_column(String(3))
    numeric_code: Mapped[str | None] = mapped_column(String(3))
    name: Mapped[str] = mapped_column(String(100))
    symbol: Mapped[str | None] = mapped_column(String(10))

    minor_unit: Mapped[int] = mapped_column(Integer, nullable=False)
    cash_rounding_increment: Mapped[int | None] = mapped_column(Integer)
    rounding_mode: Mapped[str] = mapped_column(String(30))

    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    __table_args__ = (
        Index("uq_license_currency_code", "code", unique=True, postgresql_where=text("status <> 'DELETED'")),
        CheckConstraint("minor_unit >= 0", name="ck_license_currency_minor_unit"),
    )
```

Money should be represented using integer minor units or `Decimal`, never binary floating point.

---

# 7. Product Catalog

## `license_product`

```python
class LicenseProduct(GlobalEntity):
    __tablename__ = "license_product"

    product_code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))
    display_name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    product_type: Mapped[ProductType]
    category_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product_category.id", ondelete="RESTRICT"),
    )

    is_subscription: Mapped[bool] = mapped_column(default=False)
    is_addon: Mapped[bool] = mapped_column(default=False)
    is_marketplace_app: Mapped[bool] = mapped_column(default=False)

    status: Mapped[Status]
    effective_from: Mapped[datetime | None]
    effective_to: Mapped[datetime | None]

    __table_args__ = (
        Index("uq_license_product_code", "product_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

## `license_product_category`

```python
class LicenseProductCategory(GlobalEntity):
    __tablename__ = "license_product_category"

    parent_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product_category.id", ondelete="RESTRICT"),
    )
    code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    sort_order: Mapped[int] = mapped_column(Integer, default=0)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    __table_args__ = (
        Index("uq_license_product_category_code", "code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

## `license_product_version`

```python
class LicenseProductVersion(GlobalEntity):
    __tablename__ = "license_product_version"

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"), index=True,
    )
    version_number: Mapped[int]
    version_code: Mapped[str] = mapped_column(String(50))
    description: Mapped[str | None] = mapped_column(Text)

    release_date: Mapped[datetime | None]
    effective_from: Mapped[datetime]
    effective_to: Mapped[datetime | None]

    status: Mapped[Status]

    __table_args__ = (
        UniqueConstraint("product_id", "version_number", name="uq_license_product_version"),
    )
```

---

# 8. Feature Catalog

## `license_feature`

```python
class LicenseFeature(GlobalEntity):
    __tablename__ = "license_feature"

    feature_code: Mapped[str] = mapped_column(String(150))
    name: Mapped[str] = mapped_column(String(200))
    display_name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    feature_type: Mapped[FeatureType]

    service_code: Mapped[str | None] = mapped_column(String(100))
    resource_code: Mapped[str | None] = mapped_column(String(100))

    is_metered: Mapped[bool] = mapped_column(default=False)
    is_billable: Mapped[bool] = mapped_column(default=False)
    is_enforceable: Mapped[bool] = mapped_column(default=True)

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_feature_code", "feature_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

## `license_feature_dependency`

```python
class LicenseFeatureDependency(GlobalEntity):
    __tablename__ = "license_feature_dependency"

    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="CASCADE"), index=True,
    )
    depends_on_feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    dependency_type: Mapped[str] = mapped_column(String(30))

    __table_args__ = (
        UniqueConstraint("feature_id", "depends_on_feature_id", name="uq_license_feature_dependency"),
        CheckConstraint("feature_id <> depends_on_feature_id", name="ck_license_feature_dependency_self"),
    )
```

---

# 9. Modules

```python
class LicenseModule(GlobalEntity):
    __tablename__ = "license_module"

    module_code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))
    display_name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    service_code: Mapped[str] = mapped_column(String(100))
    api_namespace: Mapped[str | None] = mapped_column(String(100))

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_module_code", "module_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseModuleFeature(GlobalEntity):
    __tablename__ = "license_module_feature"

    module_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_module.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )
    is_required: Mapped[bool] = mapped_column(default=False)

    __table_args__ = (
        UniqueConstraint("module_id", "feature_id", name="uq_license_module_feature"),
    )
```

---

# 10. Units

```python
class LicenseUnit(GlobalEntity):
    __tablename__ = "license_unit"

    code: Mapped[str] = mapped_column(String(50))
    name: Mapped[str] = mapped_column(String(100))
    symbol: Mapped[str | None] = mapped_column(String(20))

    unit_type: Mapped[str] = mapped_column(String(30))
    precision: Mapped[int] = mapped_column(Integer, default=0)

    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    __table_args__ = (
        Index("uq_license_unit_code", "code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

Examples:

```text
USER
SEAT
COMPANY
BRANCH
EMPLOYEE
API_CALL
GB
DOCUMENT
TRANSACTION
DEVICE
```

---

# 11. Pricing

## `license_price`

```python
class LicensePrice(GlobalEntity):
    __tablename__ = "license_price"

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"), index=True,
    )
    # A price may bind to a specific plan version or add-on version (matches the API filters).
    plan_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"), index=True,
    )
    addon_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_addon_version.id", ondelete="RESTRICT"), index=True,
    )

    price_code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    pricing_model: Mapped[PricingModel]

    billing_interval: Mapped[BillingInterval]
    billing_interval_count: Mapped[int] = mapped_column(Integer, default=1)

    amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)
    unit_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_unit.id", ondelete="RESTRICT"),
    )

    tax_behavior: Mapped[str | None] = mapped_column(String(30))
    tax_code: Mapped[str | None] = mapped_column(String(50))

    effective_from: Mapped[datetime]
    effective_to: Mapped[datetime | None]

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_price_code", "price_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
        CheckConstraint("effective_to IS NULL OR effective_to >= effective_from", name="ck_license_price_effective"),
    )
```

## `license_price_tier`

```python
class LicensePriceTier(GlobalEntity):
    __tablename__ = "license_price_tier"

    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="CASCADE"), index=True,
    )

    tier_number: Mapped[int]
    from_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)
    to_quantity: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    unit_amount: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    flat_amount: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    tier_type: Mapped[str] = mapped_column(String(30))

    __table_args__ = (
        UniqueConstraint("price_id", "tier_number", name="uq_license_price_tier"),
        CheckConstraint("to_quantity IS NULL OR to_quantity >= from_quantity", name="ck_license_price_tier_range"),
    )
```

## `license_price_component`

```python
class LicensePriceComponent(GlobalEntity):
    __tablename__ = "license_price_component"

    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="CASCADE"), index=True,
    )
    component_code: Mapped[str] = mapped_column(String(100))
    component_type: Mapped[str] = mapped_column(String(30))

    amount: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    percentage: Mapped[Decimal | None] = mapped_column(Numeric(9, 6))

    unit_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_unit.id", ondelete="RESTRICT"),
    )
    calculation_order: Mapped[int] = mapped_column(Integer, default=0)
```

---

# 12. Plans

## `license_plan`

```python
class LicensePlan(GlobalEntity):
    __tablename__ = "license_plan"

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"), index=True,
    )
    plan_code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))
    display_name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    plan_type: Mapped[PlanType]
    visibility: Mapped[str] = mapped_column(String(30))

    is_public: Mapped[bool] = mapped_column(default=True)
    is_custom: Mapped[bool] = mapped_column(default=False)

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_plan_code", "plan_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

## `license_plan_version`

```python
class LicensePlanVersion(GlobalEntity):
    __tablename__ = "license_plan_version"

    plan_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan.id", ondelete="RESTRICT"), index=True,
    )

    version_number: Mapped[int]
    version_code: Mapped[str] = mapped_column(String(50))

    effective_from: Mapped[datetime]
    effective_to: Mapped[datetime | None]

    status: Mapped[Status]

    published_at: Mapped[datetime | None]
    published_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    __table_args__ = (
        UniqueConstraint("plan_id", "version_number", name="uq_license_plan_version"),
    )
```

Plan versions are immutable after publication.

## `license_plan_version_feature`

```python
class LicensePlanVersionFeature(GlobalEntity):
    __tablename__ = "license_plan_version_feature"

    plan_version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    enabled: Mapped[bool] = mapped_column(default=False)

    value_type: Mapped[str] = mapped_column(String(30))
    value_numeric: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    value_boolean: Mapped[bool | None]
    value_text: Mapped[str | None]
    value_json: Mapped[dict | None]

    unit_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_unit.id", ondelete="RESTRICT"),
    )

    __table_args__ = (
        UniqueConstraint("plan_version_id", "feature_id", name="uq_license_plan_version_feature"),
    )
```

---

# 13. Add-ons

```python
class LicenseAddon(GlobalEntity):
    __tablename__ = "license_addon"

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"), index=True,
    )
    addon_code: Mapped[str] = mapped_column(String(100))

    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    addon_type: Mapped[str] = mapped_column(String(30))
    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_addon_code", "addon_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseAddonVersion(GlobalEntity):
    __tablename__ = "license_addon_version"

    addon_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_addon.id", ondelete="RESTRICT"), index=True,
    )
    version_number: Mapped[int]

    effective_from: Mapped[datetime]
    effective_to: Mapped[datetime | None]

    status: Mapped[Status]

    __table_args__ = (
        UniqueConstraint("addon_id", "version_number", name="uq_license_addon_version"),
    )


class LicenseAddonFeature(GlobalEntity):
    __tablename__ = "license_addon_feature"

    addon_version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_addon_version.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    value_type: Mapped[str] = mapped_column(String(30))
    value_numeric: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    value_boolean: Mapped[bool | None]
    value_json: Mapped[dict | None]

    __table_args__ = (
        UniqueConstraint("addon_version_id", "feature_id", name="uq_license_addon_feature"),
    )
```

---

# 14. Subscription

## `license_subscription`

```python
class LicenseSubscription(TenantScopedEntity):
    __tablename__ = "license_subscription"

    subscription_number: Mapped[str] = mapped_column(String(100))

    plan_version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"), index=True,
    )
    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    status: Mapped[SubscriptionStatus] = mapped_column(index=True)

    started_at: Mapped[datetime]
    trial_start_at: Mapped[datetime | None]
    trial_end_at: Mapped[datetime | None]

    current_period_start: Mapped[datetime]
    current_period_end: Mapped[datetime] = mapped_column(index=True)

    billing_anchor: Mapped[int | None]

    billing_interval: Mapped[BillingInterval]
    billing_interval_count: Mapped[int] = mapped_column(Integer, default=1)

    cancel_at_period_end: Mapped[bool] = mapped_column(default=False)
    cancelled_at: Mapped[datetime | None]
    cancellation_reason: Mapped[str | None]

    paused_at: Mapped[datetime | None]
    resume_at: Mapped[datetime | None]

    auto_renew: Mapped[bool] = mapped_column(default=True)
    collection_method: Mapped[str] = mapped_column(String(30))

    billing_account_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_billing_account.id", ondelete="RESTRICT"),
    )
    contract_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_contract.id", ondelete="RESTRICT"),
    )

    __table_args__ = (
        # Subscription number is unique per tenant (not global) — §64.1.
        Index("uq_license_subscription_number", "tenant_id", "subscription_number",
              unique=True, postgresql_where=text("status <> 'DELETED'")),
        Index("ix_license_subscription_tenant_status", "tenant_id", "status"),
    )
```

---

# 15. Subscription Items

```python
class LicenseSubscriptionItem(TenantScopedEntity):
    __tablename__ = "license_subscription_item"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"),
    )
    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )

    plan_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )
    addon_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_addon_version.id", ondelete="RESTRICT"),
    )

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)

    unit_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)
    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    billing_interval: Mapped[BillingInterval]
    billing_interval_count: Mapped[int] = mapped_column(Integer, default=1)

    current_period_start: Mapped[datetime]
    current_period_end: Mapped[datetime]

    started_at: Mapped[datetime]
    ended_at: Mapped[datetime | None]

    status: Mapped[Status]

    __table_args__ = (
        Index("ix_license_subscription_item_sub", "subscription_id", "status"),
    )
```

---

# 16. Subscription Changes

```python
class LicenseSubscriptionChange(TenantScopedEntity):
    __tablename__ = "license_subscription_change"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )

    change_type: Mapped[SubscriptionChangeType]
    effective_type: Mapped[EffectiveType]
    effective_at: Mapped[datetime]

    old_plan_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )
    new_plan_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )

    old_quantity: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    new_quantity: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    old_price_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )
    new_price_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )

    proration_mode: Mapped[str | None] = mapped_column(String(30))
    proration_amount: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    status: Mapped[str] = mapped_column(String(30))

    requested_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))
    approved_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    reason: Mapped[str | None]

    executed_at: Mapped[datetime | None]

    __table_args__ = (
        Index("ix_license_subscription_change_sub", "subscription_id", "effective_at"),
    )
```

---

# 17. Subscription Schedule

```python
class LicenseSubscriptionSchedule(TenantScopedEntity):
    __tablename__ = "license_subscription_schedule"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )

    status: Mapped[str] = mapped_column(String(30))

    start_at: Mapped[datetime]
    end_at: Mapped[datetime | None]

    # No FK (would be circular with phase); validated at the service layer.
    current_phase_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))


class LicenseSubscriptionSchedulePhase(TenantScopedEntity):
    __tablename__ = "license_subscription_schedule_phase"

    schedule_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription_schedule.id", ondelete="CASCADE"), index=True,
    )

    phase_number: Mapped[int]

    start_at: Mapped[datetime]
    end_at: Mapped[datetime | None]

    plan_version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )

    billing_interval: Mapped[BillingInterval]
    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    __table_args__ = (
        UniqueConstraint("schedule_id", "phase_number", name="uq_license_schedule_phase"),
    )
```

---

# 18. Trials

```python
class LicenseTrialPolicy(GlobalEntity):
    __tablename__ = "license_trial_policy"

    name: Mapped[str] = mapped_column(String(200))
    duration_days: Mapped[int]

    requires_payment_method: Mapped[bool] = mapped_column(default=False)
    auto_convert: Mapped[bool] = mapped_column(default=False)

    conversion_plan_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )

    status: Mapped[Status]

    __table_args__ = (
        CheckConstraint("duration_days > 0", name="ck_license_trial_policy_days"),
    )


class LicenseSubscriptionTrial(TenantScopedEntity):
    __tablename__ = "license_subscription_trial"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    trial_policy_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_trial_policy.id", ondelete="RESTRICT"),
    )

    started_at: Mapped[datetime]
    ends_at: Mapped[datetime]

    converted_at: Mapped[datetime | None]
    cancelled_at: Mapped[datetime | None]

    status: Mapped[str] = mapped_column(String(30))
```

---

# 19. Contracts

```python
class LicenseContract(TenantScopedEntity):
    __tablename__ = "license_contract"

    contract_number: Mapped[str] = mapped_column(String(100))

    start_date: Mapped[datetime]
    end_date: Mapped[datetime | None]

    auto_renew: Mapped[bool] = mapped_column(default=False)
    notice_period_days: Mapped[int] = mapped_column(Integer, default=0)

    minimum_commitment: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    committed_amount: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    currency_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_contract_number", "tenant_id", "contract_number",
              unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseContractLine(TenantScopedEntity):
    __tablename__ = "license_contract_line"

    contract_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_contract.id", ondelete="CASCADE"), index=True,
    )

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"),
    )
    plan_version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )
    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )

    committed_quantity: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    minimum_quantity: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    unit_price: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    billing_interval: Mapped[BillingInterval]
```

---

# 20. Entitlement Engine

## `license_entitlement`

```python
class LicenseEntitlement(TenantScopedEntity):
    __tablename__ = "license_entitlement"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )

    entitlement_version: Mapped[int]

    status: Mapped[Status]

    valid_from: Mapped[datetime]
    valid_until: Mapped[datetime | None]

    compiled_at: Mapped[datetime]
    compiled_by: Mapped[str] = mapped_column(String(100))

    source_hash: Mapped[str] = mapped_column(String(128))
    payload_hash: Mapped[str] = mapped_column(String(128))

    jwt_id: Mapped[str | None] = mapped_column(String(100))
    jwt_key_id: Mapped[str | None] = mapped_column(String(100))

    __table_args__ = (
        UniqueConstraint("subscription_id", "entitlement_version", name="uq_license_entitlement_version"),
        Index("ix_license_entitlement_tenant_status", "tenant_id", "status"),
    )
```

## `license_entitlement_feature`

```python
# Child of a tenant-scoped entitlement; tenant scope flows through entitlement_id.
class LicenseEntitlementFeature(GlobalEntity):
    __tablename__ = "license_entitlement_feature"

    entitlement_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_entitlement.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    enabled: Mapped[bool] = mapped_column(default=False)

    value_type: Mapped[str] = mapped_column(String(30))
    value_numeric: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    value_boolean: Mapped[bool | None]
    value_text: Mapped[str | None]
    value_json: Mapped[dict | None]

    # Entitlement provenance (folded — no separate license_entitlement_source table, §63.1).
    source_type: Mapped[str] = mapped_column(String(30))
    source_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    effective_from: Mapped[datetime]
    effective_until: Mapped[datetime | None]

    __table_args__ = (
        UniqueConstraint("entitlement_id", "feature_id", name="uq_license_entitlement_feature"),
    )
```

## `license_entitlement_limit`

```python
class LicenseEntitlementLimit(GlobalEntity):
    __tablename__ = "license_entitlement_limit"

    entitlement_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_entitlement.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    limit_type: Mapped[LimitType]

    limit_value: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)
    unit_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_unit.id", ondelete="RESTRICT"),
    )

    period_type: Mapped[str | None] = mapped_column(String(30))
    period_seconds: Mapped[int | None]

    hard_limit: Mapped[bool] = mapped_column(default=True)
    soft_limit: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    warning_threshold: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    __table_args__ = (
        UniqueConstraint("entitlement_id", "feature_id", "limit_type", name="uq_license_entitlement_limit"),
    )
```

---

# 21. Entitlement Overrides

```python
class LicenseEntitlementOverride(TenantScopedEntity):
    __tablename__ = "license_entitlement_override"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    override_type: Mapped[str] = mapped_column(String(30))

    value_numeric: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    value_boolean: Mapped[bool | None]
    value_json: Mapped[dict | None]

    reason: Mapped[str]

    effective_from: Mapped[datetime]
    effective_until: Mapped[datetime | None]

    approved_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))
```

---

# 22. Entitlement Compilation

```python
class LicenseEntitlementCompilation(GlobalEntity):
    __tablename__ = "license_entitlement_compilation"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )

    source_version: Mapped[int]
    target_version: Mapped[int]

    status: Mapped[str] = mapped_column(String(30))

    started_at: Mapped[datetime | None]
    completed_at: Mapped[datetime | None]

    source_hash: Mapped[str | None] = mapped_column(String(128))
    result_hash: Mapped[str | None] = mapped_column(String(128))

    error_code: Mapped[str | None] = mapped_column(String(100))
    error_message: Mapped[str | None]
```

---

# 23. Entitlement Token

```python
class LicenseEntitlementToken(GlobalEntity):
    __tablename__ = "license_entitlement_token"

    tenant_id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), index=True)  # logical ref
    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    entitlement_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_entitlement.id", ondelete="CASCADE"),
    )

    jti: Mapped[str] = mapped_column(String(100))
    key_id: Mapped[str] = mapped_column(String(100))

    token_hash: Mapped[str] = mapped_column(String(128))

    issued_at: Mapped[datetime]
    expires_at: Mapped[datetime]

    revoked_at: Mapped[datetime | None]

    status: Mapped[str] = mapped_column(String(30))

    __table_args__ = (
        Index("uq_license_entitlement_token_jti", "jti", unique=True),
        Index("ix_license_entitlement_token_sub", "subscription_id", "status"),
    )
```

---

# 24. Signing Keys

```python
class LicenseSigningKey(GlobalEntity):
    __tablename__ = "license_signing_key"

    key_id: Mapped[str] = mapped_column(String(100))

    algorithm: Mapped[str] = mapped_column(String(30))
    key_type: Mapped[str] = mapped_column(String(30))

    public_key: Mapped[str] = mapped_column(Text)

    # Reference into KMS/HSM/secret manager — never the raw private key.
    private_key_reference: Mapped[str] = mapped_column(String(255))

    valid_from: Mapped[datetime]
    valid_until: Mapped[datetime | None]

    rotation_version: Mapped[int]

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_signing_key_id", "key_id", unique=True),
    )
```

Private keys must be stored in a KMS/HSM/secret manager, not PostgreSQL.

---

# 25. Metering

```python
class LicenseMeter(GlobalEntity):
    __tablename__ = "license_meter"

    meter_code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    event_name: Mapped[str] = mapped_column(String(150))
    aggregation_type: Mapped[AggregationType]

    unit_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_unit.id", ondelete="RESTRICT"),
    )

    reset_period: Mapped[str | None] = mapped_column(String(30))

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_meter_code", "meter_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

---

# 26. Usage Events

```python
class LicenseMeterEvent(TenantScopedEntity):
    __tablename__ = "license_meter_event"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    meter_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_meter.id", ondelete="RESTRICT"), index=True,
    )

    event_id: Mapped[str] = mapped_column(String(100))
    event_type: Mapped[str] = mapped_column(String(100))

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)

    resource_id: Mapped[str | None] = mapped_column(String(255))
    resource_type: Mapped[str | None] = mapped_column(String(100))

    occurred_at: Mapped[datetime]
    recorded_at: Mapped[datetime]

    idempotency_key: Mapped[str] = mapped_column(String(255))

    __table_args__ = (
        # Dedup ingestion: at most one event per (tenant, meter, idempotency_key).
        UniqueConstraint("tenant_id", "meter_id", "idempotency_key", name="uq_license_meter_event_idem"),
        Index("ix_license_meter_event_occurred", "tenant_id", "meter_id", "occurred_at"),
    )
```

---

# 27. Usage Period

```python
class LicenseUsagePeriod(TenantScopedEntity):
    __tablename__ = "license_usage_period"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    meter_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_meter.id", ondelete="RESTRICT"), index=True,
    )

    period_start: Mapped[datetime]
    period_end: Mapped[datetime]

    included_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    consumed_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    overage_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    status: Mapped[str] = mapped_column(String(30))

    __table_args__ = (
        Index("ix_license_usage_period_range", "tenant_id", "period_start", "period_end"),
    )
```

---

# 28. Usage State

```python
class LicenseUsage(TenantScopedEntity):
    __tablename__ = "license_usage"

    usage_period_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_usage_period.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    current_value: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    reserved_value: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    consumed_value: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    limit_value: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    remaining_value: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    # Concurrency uses the base-class `version` (optimistic lock); no separate
    # row_version. The usage counter must be updated under that version guard.
    last_event_at: Mapped[datetime | None]

    __table_args__ = (
        UniqueConstraint("usage_period_id", "feature_id", name="uq_license_usage_period_feature"),
    )
```

---

# 29. Immutable Usage Ledger

```python
class LicenseUsageLedger(TenantScopedEntity):
    __tablename__ = "license_usage_ledger"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    meter_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_meter.id", ondelete="RESTRICT"), index=True,
    )

    event_id: Mapped[str] = mapped_column(String(100))

    entry_type: Mapped[EntryType]

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)

    balance_before: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    balance_after: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    occurred_at: Mapped[datetime]

    __table_args__ = (
        Index("ix_license_usage_ledger_sub", "tenant_id", "subscription_id", "meter_id", "occurred_at"),
    )
```

This is a metering ledger, NOT an accounting ledger. It is append-only (§64) and a high-volume partition candidate (§66).

---

# 30. Usage Rating

```python
class LicenseUsageOverage(TenantScopedEntity):
    __tablename__ = "license_usage_overage"

    usage_period_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_usage_period.id", ondelete="CASCADE"), index=True,
    )
    meter_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_meter.id", ondelete="RESTRICT"),
    )

    included_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    actual_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    overage_quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )

    rated_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    status: Mapped[str] = mapped_column(String(30))


class LicenseUsageRating(TenantScopedEntity):
    __tablename__ = "license_usage_rating"

    usage_overage_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_usage_overage.id", ondelete="CASCADE"), index=True,
    )
    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    unit_price: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    gross_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    discount_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    net_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    calculation_version: Mapped[str] = mapped_column(String(50))

    rated_at: Mapped[datetime]
```

---

# 31. Quotas and Rate Limits

```python
class LicenseRateLimitPolicy(GlobalEntity):
    __tablename__ = "license_rate_limit_policy"

    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="CASCADE"), index=True,
    )

    scope_type: Mapped[str] = mapped_column(String(30))
    scope_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    requests: Mapped[int]
    window_seconds: Mapped[int]

    burst: Mapped[int | None]

    algorithm: Mapped[str] = mapped_column(String(30))

    status: Mapped[Status]

    __table_args__ = (
        CheckConstraint("requests > 0 AND window_seconds > 0", name="ck_license_rate_limit_positive"),
    )
```

Recommended algorithms:

```text
TOKEN_BUCKET
SLIDING_WINDOW
FIXED_WINDOW
LEAKY_BUCKET
```

---

# 32. Checkout

```python
class LicenseCheckout(TenantScopedEntity):
    __tablename__ = "license_checkout"

    checkout_number: Mapped[str] = mapped_column(String(100))

    billing_account_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_billing_account.id", ondelete="RESTRICT"), index=True,
    )

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    status: Mapped[str] = mapped_column(String(30))

    expires_at: Mapped[datetime]

    subtotal: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    discount_total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax_total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    coupon_code: Mapped[str | None] = mapped_column(String(100))

    completed_at: Mapped[datetime | None]

    __table_args__ = (
        Index("uq_license_checkout_number", "tenant_id", "checkout_number",
              unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseCheckoutItem(TenantScopedEntity):
    __tablename__ = "license_checkout_item"

    checkout_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_checkout.id", ondelete="CASCADE"), index=True,
    )

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"),
    )
    plan_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )
    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )
    addon_version_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_addon_version.id", ondelete="RESTRICT"),
    )

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    unit_price: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    subtotal: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    discount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
```

---

# 33. Quotes

```python
class LicenseQuote(TenantScopedEntity):
    __tablename__ = "license_quote"

    quote_number: Mapped[str] = mapped_column(String(100))

    billing_account_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_billing_account.id", ondelete="RESTRICT"), index=True,
    )

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    status: Mapped[str] = mapped_column(String(30))

    valid_until: Mapped[datetime]

    subtotal: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    discount_total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax_total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    accepted_at: Mapped[datetime | None]
    rejected_at: Mapped[datetime | None]

    __table_args__ = (
        Index("uq_license_quote_number", "tenant_id", "quote_number",
              unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseQuoteLine(TenantScopedEntity):
    __tablename__ = "license_quote_line"

    quote_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_quote.id", ondelete="CASCADE"), index=True,
    )

    product_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"),
    )
    plan_version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )
    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    unit_price: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    discount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
```

---

# 34. Discounts and Coupons

```python
class LicenseDiscount(GlobalEntity):
    __tablename__ = "license_discount"

    discount_code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))

    discount_type: Mapped[str] = mapped_column(String(30))

    percentage: Mapped[Decimal | None] = mapped_column(Numeric(9, 6))
    fixed_amount: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    currency_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    duration_type: Mapped[str] = mapped_column(String(30))
    duration_cycles: Mapped[int | None]

    max_discount_amount: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))

    effective_from: Mapped[datetime]
    effective_until: Mapped[datetime | None]

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_discount_code", "discount_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseCoupon(GlobalEntity):
    __tablename__ = "license_coupon"

    code: Mapped[str] = mapped_column(String(100))

    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    discount_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_discount.id", ondelete="RESTRICT"), index=True,
    )

    redemption_limit: Mapped[int | None]
    redemption_count: Mapped[int] = mapped_column(Integer, default=0)

    per_customer_limit: Mapped[int | None]

    first_time_customer_only: Mapped[bool] = mapped_column(default=False)

    valid_from: Mapped[datetime]
    valid_until: Mapped[datetime | None]

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_coupon_code", "code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseCouponCondition(GlobalEntity):
    __tablename__ = "license_coupon_condition"

    coupon_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_coupon.id", ondelete="CASCADE"), index=True,
    )

    condition_type: Mapped[str] = mapped_column(String(30))
    operator: Mapped[str] = mapped_column(String(20))

    value: Mapped[str | None]
    value_json: Mapped[dict | None]

    priority: Mapped[int] = mapped_column(Integer, default=0)


class LicenseCouponUsage(TenantScopedEntity):
    __tablename__ = "license_coupon_usage"

    coupon_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_coupon.id", ondelete="RESTRICT"), index=True,
    )
    subscription_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="SET NULL"),
    )

    discount_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    redeemed_at: Mapped[datetime]
```

---

# 35. Billing Coordination

Billing here means subscription-cycle coordination, not accounting.

```python
class LicenseBillingAccount(TenantScopedEntity):
    __tablename__ = "license_billing_account"

    customer_reference: Mapped[str | None] = mapped_column(String(255))

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    billing_email: Mapped[str] = mapped_column(String(255))
    billing_name: Mapped[str] = mapped_column(String(255))

    tax_profile_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    payment_customer_external_ref: Mapped[str | None] = mapped_column(String(255))

    collection_method: Mapped[str] = mapped_column(String(30))
    payment_terms_days: Mapped[int] = mapped_column(Integer, default=0)

    status: Mapped[Status]


class LicenseBillingCycle(TenantScopedEntity):
    __tablename__ = "license_billing_cycle"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )

    period_start: Mapped[datetime]
    period_end: Mapped[datetime]

    billing_date: Mapped[datetime]

    status: Mapped[str] = mapped_column(String(30))

    invoice_reference: Mapped[str | None] = mapped_column(String(100))
```

---

# 36. Invoice Calculation Record

Do not turn Licensing into an accounting system. Store the commercial invoice/calculation representation needed to communicate with a dedicated billing/payment/accounting platform.

```python
class LicenseInvoice(TenantScopedEntity):
    __tablename__ = "license_invoice"

    invoice_number: Mapped[str] = mapped_column(String(100))

    billing_account_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_billing_account.id", ondelete="RESTRICT"), index=True,
    )
    subscription_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="RESTRICT"), index=True,
    )

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    status: Mapped[str] = mapped_column(String(30), index=True)

    issue_date: Mapped[datetime]
    due_date: Mapped[datetime]

    period_start: Mapped[datetime]
    period_end: Mapped[datetime]

    subtotal: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    discount_total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax_total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    external_invoice_reference: Mapped[str | None]

    finalized_at: Mapped[datetime | None]

    __table_args__ = (
        # Invoice number unique per tenant (§64.1).
        Index("uq_license_invoice_number", "tenant_id", "invoice_number",
              unique=True, postgresql_where=text("status <> 'DELETED'")),
        Index("ix_license_invoice_tenant_status", "tenant_id", "status"),
    )
```

---

# 37. Invoice Lines

```python
class LicenseInvoiceLine(TenantScopedEntity):
    __tablename__ = "license_invoice_line"

    invoice_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_invoice.id", ondelete="CASCADE"), index=True,
    )

    subscription_item_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription_item.id", ondelete="SET NULL"),
    )

    product_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product.id", ondelete="RESTRICT"),
    )
    price_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )
    meter_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_meter.id", ondelete="RESTRICT"),
    )

    description: Mapped[str] = mapped_column(String(500))

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    unit_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    subtotal: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    discount_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    service_period_start: Mapped[datetime | None]
    service_period_end: Mapped[datetime | None]

    proration: Mapped[bool] = mapped_column(default=False)
    usage_based: Mapped[bool] = mapped_column(default=False)

    tax_code: Mapped[str | None] = mapped_column(String(50))
```

---

# 38. Payment References

Payment processing belongs to the dedicated Payment Platform.

Licensing only stores the integration state/reference required for subscription lifecycle.

```python
class LicensePaymentReference(TenantScopedEntity):
    __tablename__ = "license_payment_reference"

    invoice_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_invoice.id", ondelete="RESTRICT"), index=True,
    )
    subscription_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="RESTRICT"),
    )

    provider: Mapped[str] = mapped_column(String(50))
    external_payment_id: Mapped[str] = mapped_column(String(255))

    amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    status: Mapped[str] = mapped_column(String(30))

    succeeded_at: Mapped[datetime | None]
    failed_at: Mapped[datetime | None]

    failure_code: Mapped[str | None] = mapped_column(String(100))

    __table_args__ = (
        Index("uq_license_payment_reference_ext", "provider", "external_payment_id", unique=True),
    )
```

---

# 39. Refund / Chargeback References

```python
class LicenseRefundReference(TenantScopedEntity):
    __tablename__ = "license_refund_reference"

    payment_external_reference: Mapped[str] = mapped_column(String(255))

    provider: Mapped[str] = mapped_column(String(50))
    external_refund_id: Mapped[str] = mapped_column(String(255))

    amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    reason: Mapped[str | None]
    status: Mapped[str] = mapped_column(String(30))

    completed_at: Mapped[datetime | None]

    __table_args__ = (
        Index("uq_license_refund_reference_ext", "provider", "external_refund_id", unique=True),
    )


class LicenseChargebackReference(TenantScopedEntity):
    __tablename__ = "license_chargeback_reference"

    payment_external_reference: Mapped[str] = mapped_column(String(255))

    provider: Mapped[str] = mapped_column(String(50))
    external_dispute_id: Mapped[str] = mapped_column(String(255))

    amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    reason: Mapped[str | None]
    status: Mapped[str] = mapped_column(String(30))

    opened_at: Mapped[datetime | None]
    resolved_at: Mapped[datetime | None]

    __table_args__ = (
        Index("uq_license_chargeback_reference_ext", "provider", "external_dispute_id", unique=True),
    )
```

---

# 40. Proration

```python
class LicenseProration(TenantScopedEntity):
    __tablename__ = "license_proration"

    subscription_change_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription_change.id", ondelete="CASCADE"), index=True,
    )

    old_item_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription_item.id", ondelete="SET NULL"),
    )
    new_item_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription_item.id", ondelete="SET NULL"),
    )

    period_start: Mapped[datetime]
    period_end: Mapped[datetime]

    unused_fraction: Mapped[Decimal] = mapped_column(Numeric(9, 6))

    credit_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    charge_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    net_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    calculation_version: Mapped[str] = mapped_column(String(50))
```

---

# 41. Dunning

Dunning state relevant to subscription lifecycle belongs here. Payment retry execution belongs to the Payment Platform.

```python
class LicenseDunningPolicy(GlobalEntity):
    __tablename__ = "license_dunning_policy"

    name: Mapped[str] = mapped_column(String(200))

    retry_count: Mapped[int]
    retry_interval: Mapped[int]

    grace_period_days: Mapped[int]

    suspend_after_days: Mapped[int]
    cancel_after_days: Mapped[int]

    status: Mapped[Status]


class LicenseDunningCase(TenantScopedEntity):
    __tablename__ = "license_dunning_case"

    billing_account_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_billing_account.id", ondelete="RESTRICT"), index=True,
    )
    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    invoice_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_invoice.id", ondelete="SET NULL"),
    )

    status: Mapped[str] = mapped_column(String(30))

    opened_at: Mapped[datetime]
    resolved_at: Mapped[datetime | None]
```

---

# 42. Customer Credit / Balance

If credits are commercial subscription credits, Licensing can maintain them. Accounting treatment remains external.

```python
class LicenseCustomerBalance(TenantScopedEntity):
    __tablename__ = "license_customer_balance"

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    available_balance: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    reserved_balance: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    # `version` (optimistic lock) comes from the base class — do not redeclare.


class LicenseBalanceTransaction(TenantScopedEntity):
    __tablename__ = "license_balance_transaction"

    transaction_type: Mapped[str] = mapped_column(String(30))

    amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    balance_before: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    balance_after: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    source_type: Mapped[str] = mapped_column(String(30))
    source_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    description: Mapped[str | None]
```

---

# 43. Marketplace

```python
class LicenseMarketplacePublisher(GlobalEntity):
    __tablename__ = "license_marketplace_publisher"

    publisher_code: Mapped[str] = mapped_column(String(100))
    name: Mapped[str] = mapped_column(String(200))
    legal_name: Mapped[str | None] = mapped_column(String(255))

    website: Mapped[str | None] = mapped_column(String(255))

    verification_status: Mapped[str] = mapped_column(String(30))
    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_marketplace_publisher_code", "publisher_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )


class LicenseMarketplaceApp(GlobalEntity):
    __tablename__ = "license_marketplace_app"

    publisher_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_marketplace_publisher.id", ondelete="RESTRICT"), index=True,
    )

    app_code: Mapped[str] = mapped_column(String(100))

    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)

    category_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_product_category.id", ondelete="RESTRICT"),
    )

    visibility: Mapped[str] = mapped_column(String(30))
    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_marketplace_app_code", "app_code", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

---

# 44. Marketplace Versions and Artifacts

```python
class LicenseMarketplaceVersion(GlobalEntity):
    __tablename__ = "license_marketplace_version"

    app_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_marketplace_app.id", ondelete="CASCADE"), index=True,
    )

    # `version` here is the marketplace artifact's semantic version (String),
    # distinct from the base optimistic-lock integer `version`.
    app_version: Mapped[str] = mapped_column(String(50))
    release_notes: Mapped[str | None] = mapped_column(Text)

    artifact_url: Mapped[str] = mapped_column(String(500))
    artifact_hash: Mapped[str] = mapped_column(String(128))
    artifact_size: Mapped[int]

    signature: Mapped[str] = mapped_column(Text)
    signing_key_id: Mapped[str] = mapped_column(String(100))

    minimum_platform_version: Mapped[str | None] = mapped_column(String(50))
    maximum_platform_version: Mapped[str | None] = mapped_column(String(50))

    status: Mapped[Status]

    published_at: Mapped[datetime | None]

    __table_args__ = (
        UniqueConstraint("app_id", "app_version", name="uq_license_marketplace_version"),
    )
```

Artifact URL alone must never be treated as trust.

Validate:

```text
hash
signature
publisher
version
permissions
compatibility
```

---

# 45. Marketplace Purchase and Installation

```python
class LicenseMarketplacePurchase(TenantScopedEntity):
    __tablename__ = "license_marketplace_purchase"

    app_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_marketplace_app.id", ondelete="RESTRICT"), index=True,
    )
    version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_marketplace_version.id", ondelete="RESTRICT"),
    )

    subscription_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="SET NULL"),
    )
    price_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_price.id", ondelete="RESTRICT"),
    )

    quantity: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    status: Mapped[str] = mapped_column(String(30))

    purchased_at: Mapped[datetime]


class LicenseMarketplaceInstallation(TenantScopedEntity):
    __tablename__ = "license_marketplace_installation"

    purchase_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_marketplace_purchase.id", ondelete="CASCADE"), index=True,
    )
    version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_marketplace_version.id", ondelete="RESTRICT"),
    )

    installation_id: Mapped[str] = mapped_column(String(100))

    status: Mapped[str] = mapped_column(String(30))

    installed_at: Mapped[datetime | None]
    uninstalled_at: Mapped[datetime | None]
```

---

# 46. Marketplace Permissions

```python
class LicenseMarketplacePermission(GlobalEntity):
    __tablename__ = "license_marketplace_permission"

    app_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_marketplace_app.id", ondelete="CASCADE"), index=True,
    )

    permission_code: Mapped[str] = mapped_column(String(100))
    resource: Mapped[str] = mapped_column(String(100))
    action: Mapped[str] = mapped_column(String(50))

    risk_level: Mapped[str] = mapped_column(String(20))
```

---

# 47. On-Premise Deployment

```python
class LicenseDeployment(TenantScopedEntity):
    __tablename__ = "license_deployment"

    deployment_code: Mapped[str] = mapped_column(String(100))
    deployment_type: Mapped[str] = mapped_column(String(30))

    environment: Mapped[str] = mapped_column(String(30))

    last_seen_at: Mapped[datetime | None]

    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_deployment_code", "tenant_id", "deployment_code",
              unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

---

# 48. License Key

```python
class LicenseLicenseKey(TenantScopedEntity):
    __tablename__ = "license_license_key"

    license_number: Mapped[str] = mapped_column(String(100))

    deployment_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_deployment.id", ondelete="SET NULL"),
    )

    license_type: Mapped[str] = mapped_column(String(30))

    issued_at: Mapped[datetime]
    valid_from: Mapped[datetime]
    valid_until: Mapped[datetime | None]

    status: Mapped[Status]

    key_id: Mapped[str] = mapped_column(String(100))

    payload_hash: Mapped[str] = mapped_column(String(128))
    signature: Mapped[str] = mapped_column(Text)

    activation_limit: Mapped[int | None]
    activation_count: Mapped[int] = mapped_column(Integer, default=0)

    __table_args__ = (
        Index("uq_license_license_number", "license_number", unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

---

# 49. Activation

```python
class LicenseActivation(TenantScopedEntity):
    __tablename__ = "license_activation"

    license_key_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_license_key.id", ondelete="CASCADE"), index=True,
    )
    deployment_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_deployment.id", ondelete="RESTRICT"),
    )

    activation_code: Mapped[str] = mapped_column(String(100))

    # Store a hash of the hardware fingerprint, never the raw identifier.
    hardware_fingerprint: Mapped[str | None] = mapped_column(String(128))

    activated_at: Mapped[datetime | None]
    deactivated_at: Mapped[datetime | None]

    status: Mapped[str] = mapped_column(String(30))
```

Store only a fingerprint hash where possible.

---

# 50. Hardware Binding

```python
class LicenseHardwareBinding(TenantScopedEntity):
    __tablename__ = "license_hardware_binding"

    deployment_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_deployment.id", ondelete="CASCADE"), index=True,
    )

    fingerprint_hash: Mapped[str] = mapped_column(String(128))

    binding_type: Mapped[str] = mapped_column(String(30))

    first_seen_at: Mapped[datetime]
    last_seen_at: Mapped[datetime]

    status: Mapped[Status]
```

---

# 51. Revocation

```python
class LicenseRevocation(TenantScopedEntity):
    __tablename__ = "license_revocation"

    license_key_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_license_key.id", ondelete="CASCADE"), index=True,
    )
    deployment_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_deployment.id", ondelete="SET NULL"),
    )

    reason: Mapped[str]

    revoked_at: Mapped[datetime]
    revoked_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    effective_at: Mapped[datetime]

    status: Mapped[str] = mapped_column(String(30))
```

---

# 52. Webhooks

```python
class LicenseWebhookEndpoint(TenantScopedEntity):
    __tablename__ = "license_webhook_endpoint"

    url: Mapped[str] = mapped_column(String(500))

    # Reference to a secret manager entry — never the raw signing secret.
    secret_reference: Mapped[str] = mapped_column(String(255))

    status: Mapped[Status]

    event_filters: Mapped[list] = mapped_column(JSONB, default=list)

    failure_count: Mapped[int] = mapped_column(Integer, default=0)

    last_success_at: Mapped[datetime | None]
    last_failure_at: Mapped[datetime | None]


class LicenseWebhookDelivery(TenantScopedEntity):
    __tablename__ = "license_webhook_delivery"

    endpoint_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_webhook_endpoint.id", ondelete="CASCADE"), index=True,
    )
    event_id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), index=True)

    attempt_number: Mapped[int]

    status: Mapped[str] = mapped_column(String(30))

    http_status: Mapped[int | None]
    response_body: Mapped[str | None]

    attempted_at: Mapped[datetime]
    next_retry_at: Mapped[datetime | None]

    __table_args__ = (
        Index("ix_license_webhook_delivery_retry", "status", "next_retry_at"),
    )
```

---

# 53. Event Store

```python
class LicenseEvent(TenantScopedEntity):
    __tablename__ = "license_event"

    aggregate_type: Mapped[str] = mapped_column(String(100))
    aggregate_id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), index=True)

    event_type: Mapped[str] = mapped_column(String(150))
    event_version: Mapped[int]

    payload: Mapped[dict] = mapped_column(JSONB)
    headers: Mapped[dict] = mapped_column(JSONB, default=dict)

    occurred_at: Mapped[datetime]

    correlation_id: Mapped[str | None] = mapped_column(String(100))
    causation_id: Mapped[str | None] = mapped_column(String(100))

    idempotency_key: Mapped[str | None] = mapped_column(String(255))

    processed_at: Mapped[datetime | None]

    __table_args__ = (
        Index("ix_license_event_type", "tenant_id", "event_type", "occurred_at"),
    )
```

---

# 54. Transactional Outbox

```python
class LicenseOutboxEvent(Base):
    __tablename__ = "license_outbox_event"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    aggregate_type: Mapped[str] = mapped_column(String(100))
    aggregate_id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True))

    event_type: Mapped[str] = mapped_column(String(150))

    payload: Mapped[dict] = mapped_column(JSONB)
    headers: Mapped[dict] = mapped_column(JSONB, default=dict)

    status: Mapped[str] = mapped_column(String(30))

    attempt_count: Mapped[int] = mapped_column(Integer, default=0)

    available_at: Mapped[datetime]
    published_at: Mapped[datetime | None]

    last_error: Mapped[str | None]

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    __table_args__ = (
        # Relay poll index: due, unpublished events.
        Index("ix_license_outbox_dispatch", "status", "available_at"),
    )
```

---

# 55. Inbox

```python
class LicenseInboxEvent(Base):
    __tablename__ = "license_inbox_event"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    event_id: Mapped[str] = mapped_column(String(100))
    event_type: Mapped[str] = mapped_column(String(150))

    source: Mapped[str] = mapped_column(String(100))

    payload_hash: Mapped[str] = mapped_column(String(128))

    processed_at: Mapped[datetime | None]

    status: Mapped[str] = mapped_column(String(30))

    error_message: Mapped[str | None]

    __table_args__ = (
        # Consumer dedup: an event from a source is processed at most once.
        UniqueConstraint("source", "event_id", name="uq_license_inbox_event"),
    )
```

---

# 56. Idempotency

```python
class LicenseIdempotencyKey(TenantScopedEntity):
    __tablename__ = "license_idempotency_key"

    key: Mapped[str] = mapped_column(String(255))
    operation: Mapped[str] = mapped_column(String(100))

    request_hash: Mapped[str] = mapped_column(String(128))

    response_code: Mapped[int | None]
    response_body: Mapped[dict | None]

    expires_at: Mapped[datetime]

    __table_args__ = (
        UniqueConstraint("tenant_id", "key", "operation", name="uq_license_idempotency_key"),
    )
```

Critical operations must require idempotency:

```text
checkout
subscription creation
subscription change
coupon redemption
usage ingestion
invoice finalization
```

---

# 57. Audit

```python
class LicenseAudit(Base):
    __tablename__ = "license_audit"

    id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    tenant_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True), index=True)

    actor_type: Mapped[str] = mapped_column(String(30))
    actor_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    action: Mapped[str] = mapped_column(String(100))

    resource_type: Mapped[str] = mapped_column(String(100))
    resource_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    before_data: Mapped[dict | None] = mapped_column(JSONB)
    after_data: Mapped[dict | None] = mapped_column(JSONB)

    ip_address: Mapped[str | None] = mapped_column(INET)
    user_agent: Mapped[str | None]

    correlation_id: Mapped[str | None] = mapped_column(String(100))

    occurred_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    __table_args__ = (
        # Append-only (§64); high-volume partition candidate (§66).
        Index("ix_license_audit_tenant_time", "tenant_id", "occurred_at"),
        Index("ix_license_audit_resource", "resource_type", "resource_id"),
    )
```

Audit rows must be append-only.

---

# 58. Notification

```python
class LicenseNotificationTemplate(GlobalEntity):
    __tablename__ = "license_notification_template"

    event_type: Mapped[str] = mapped_column(String(150))
    channel: Mapped[str] = mapped_column(String(30))
    locale: Mapped[str] = mapped_column(String(10))

    subject: Mapped[str | None] = mapped_column(String(255))
    body: Mapped[str] = mapped_column(Text)

    # optimistic-lock `version` comes from the base class
    status: Mapped[Status]

    __table_args__ = (
        UniqueConstraint("event_type", "channel", "locale", name="uq_license_notification_template"),
    )


class LicenseNotificationDelivery(TenantScopedEntity):
    __tablename__ = "license_notification_delivery"

    event_id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), index=True)
    template_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_notification_template.id", ondelete="RESTRICT"),
    )

    channel: Mapped[str] = mapped_column(String(30))
    recipient: Mapped[str] = mapped_column(String(255))

    status: Mapped[str] = mapped_column(String(30))

    sent_at: Mapped[datetime | None]
    failed_at: Mapped[datetime | None]
```

---

# 59. Subscription Snapshot

Snapshots are required to reproduce historical commercial decisions.

```python
class LicenseSubscriptionSnapshot(TenantScopedEntity):
    __tablename__ = "license_subscription_snapshot"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )

    snapshot_version: Mapped[int]

    plan_version_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_plan_version.id", ondelete="RESTRICT"),
    )

    subscription_data: Mapped[dict] = mapped_column(JSONB)
    entitlement_data: Mapped[dict] = mapped_column(JSONB)
    pricing_data: Mapped[dict] = mapped_column(JSONB)

    reason: Mapped[str]

    # created_at/version come from the base class (immutable once written, §64).

    __table_args__ = (
        UniqueConstraint("subscription_id", "snapshot_version", name="uq_license_subscription_snapshot"),
    )
```

---

# 60. Pricing Calculation

```python
class LicensePricingCalculation(TenantScopedEntity):
    __tablename__ = "license_pricing_calculation"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="CASCADE"), index=True,
    )
    invoice_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_invoice.id", ondelete="SET NULL"),
    )

    calculation_type: Mapped[str] = mapped_column(String(30))

    input_snapshot: Mapped[dict] = mapped_column(JSONB)
    calculation_steps: Mapped[list] = mapped_column(JSONB)

    calculation_version: Mapped[str] = mapped_column(String(50))

    subtotal: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    discount: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    tax: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)
    total: Mapped[Decimal] = mapped_column(Numeric(18, 6), default=0)

    currency_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_currency.id", ondelete="RESTRICT"),
    )

    calculated_at: Mapped[datetime]
```

This makes billing calculations explainable and reproducible.

---

# 61. External References

Do not scatter provider IDs across the schema.

```python
class LicenseExternalReference(GlobalEntity):
    __tablename__ = "license_external_reference"

    entity_type: Mapped[str] = mapped_column(String(100))
    entity_id: Mapped[uuid.UUID] = mapped_column(PGUUID(as_uuid=True), index=True)

    provider: Mapped[str] = mapped_column(String(50))
    external_id: Mapped[str] = mapped_column(String(255))
    external_type: Mapped[str | None] = mapped_column(String(100))

    # (metadata is inherited from the base as `metadata_`)

    __table_args__ = (
        Index("uq_license_external_reference", "entity_type", "entity_id", "provider", "external_id", unique=True),
    )
```

Examples:

```text
Stripe Customer
Razorpay Customer
Payment Invoice
Tax Transaction
Accounting Invoice
```

---

# 62. Reconciliation

Reconciliation here is integration reconciliation, not accounting reconciliation.

```python
class LicenseReconciliationRun(GlobalEntity):
    __tablename__ = "license_reconciliation_run"

    provider: Mapped[str] = mapped_column(String(50))
    run_type: Mapped[str] = mapped_column(String(30))

    period_start: Mapped[datetime]
    period_end: Mapped[datetime]

    status: Mapped[str] = mapped_column(String(30))

    started_at: Mapped[datetime]
    completed_at: Mapped[datetime | None]

    total_records: Mapped[int] = mapped_column(Integer, default=0)
    matched_records: Mapped[int] = mapped_column(Integer, default=0)
    mismatched_records: Mapped[int] = mapped_column(Integer, default=0)


class LicenseReconciliationItem(GlobalEntity):
    __tablename__ = "license_reconciliation_item"

    run_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_reconciliation_run.id", ondelete="CASCADE"), index=True,
    )

    external_id: Mapped[str] = mapped_column(String(255))
    internal_id: Mapped[str | None] = mapped_column(String(255))

    expected_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    actual_amount: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    difference: Mapped[Decimal] = mapped_column(Numeric(18, 6))

    status: Mapped[str] = mapped_column(String(30))
    resolution: Mapped[str | None]
```

---

# 63. Configuration

```python
class LicenseConfiguration(GlobalEntity):
    __tablename__ = "license_configuration"

    scope_type: Mapped[str] = mapped_column(String(30))
    scope_id: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    configuration_key: Mapped[str] = mapped_column(String(200))

    value_type: Mapped[str] = mapped_column(String(30))
    value_json: Mapped[dict] = mapped_column(JSONB)

    effective_from: Mapped[datetime | None]
    effective_until: Mapped[datetime | None]

    # optimistic-lock `version` comes from the base class
    status: Mapped[Status]

    __table_args__ = (
        Index("uq_license_configuration_scope_key", "scope_type", "scope_id", "configuration_key",
              unique=True, postgresql_where=text("status <> 'DELETED'")),
    )
```

Scope:

```text
GLOBAL
PRODUCT
PLAN
TENANT
SUBSCRIPTION
```

---

# 63.1 Additional Domain Tables (developer-guide aligned)

These tables complete the developer guide's table list. They follow the §64.1 conventions (real intra-platform FKs, `Numeric` money, `GlobalEntity`/`TenantScopedEntity` base). One guide table — `license_entitlement_source` — is intentionally **not** a table: entitlement provenance is denormalized onto `license_entitlement_feature.source_type` / `source_id` (§20), because a materialized entitlement is a hot read path and a separate source table would add a join.

```python
# Contract-granted entitlements — feed entitlement compilation (§20, precedence #4).
class LicenseContractEntitlement(TenantScopedEntity):
    __tablename__ = "license_contract_entitlement"

    contract_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True),
        ForeignKey("license_contract.id", ondelete="CASCADE"), index=True,
    )
    feature_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_feature.id", ondelete="RESTRICT"),
    )

    value_type: Mapped[str]
    value_numeric: Mapped[Decimal | None] = mapped_column(Numeric(18, 6))
    value_boolean: Mapped[bool | None]
    value_json: Mapped[dict | None]

    unit_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_unit.id", ondelete="RESTRICT"),
    )
    status: Mapped[Status]


# Subscription pause history (richer than paused_at/resume_at on the subscription row).
class LicenseSubscriptionPause(TenantScopedEntity):
    __tablename__ = "license_subscription_pause"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True),
        ForeignKey("license_subscription.id", ondelete="RESTRICT"), index=True,
    )

    paused_at: Mapped[datetime]
    resume_at: Mapped[datetime | None]
    resumed_at: Mapped[datetime | None]

    reason: Mapped[str | None]
    requested_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    status: Mapped[str]


# Cancellation record (IMMEDIATE / END_OF_TERM / SCHEDULED).
class LicenseCancellation(TenantScopedEntity):
    __tablename__ = "license_cancellation"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True),
        ForeignKey("license_subscription.id", ondelete="RESTRICT"), index=True,
    )

    effective_type: Mapped[EffectiveType]
    effective_at: Mapped[datetime]

    reason: Mapped[str | None]
    requested_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))
    canceled_at: Mapped[datetime | None]

    status: Mapped[str]


# Meter/quota reset events (§21) — audit trail of resets.
class LicenseUsageReset(TenantScopedEntity):
    __tablename__ = "license_usage_reset"

    subscription_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True),
        ForeignKey("license_subscription.id", ondelete="RESTRICT"), index=True,
    )
    meter_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_meter.id", ondelete="RESTRICT"), index=True,
    )
    usage_period_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_usage_period.id", ondelete="SET NULL"),
    )

    reset_type: Mapped[str]
    reason: Mapped[str | None]
    value_before: Mapped[Decimal] = mapped_column(Numeric(18, 6))
    reset_at: Mapped[datetime]


# Structured discount rules (beyond simple coupon conditions).
class LicenseDiscountRule(GlobalEntity):
    __tablename__ = "license_discount_rule"

    discount_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_discount.id", ondelete="CASCADE"), index=True,
    )

    rule_type: Mapped[str]
    operator: Mapped[str]
    value: Mapped[str | None]
    value_json: Mapped[dict | None]
    priority: Mapped[int]


# Quote → subscription conversion record (preserves the commercial snapshot used).
class LicenseQuoteConversion(TenantScopedEntity):
    __tablename__ = "license_quote_conversion"

    quote_id: Mapped[uuid.UUID] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_quote.id", ondelete="RESTRICT"), index=True,
    )
    subscription_id: Mapped[uuid.UUID | None] = mapped_column(
        PGUUID(as_uuid=True), ForeignKey("license_subscription.id", ondelete="SET NULL"),
    )

    converted_at: Mapped[datetime]
    converted_by: Mapped[uuid.UUID | None] = mapped_column(PGUUID(as_uuid=True))

    snapshot: Mapped[dict]   # immutable commercial snapshot used for conversion
    status: Mapped[str]
```

---

# 64. Database Rules

## Money

Never:

```python
float
```

Use:

```python
Decimal
```

or integer minor units.

## Immutable records

The following should never be updated after finalization:

```text
Published plan version
Published price version
Finalized invoice calculation
Usage ledger entry
Audit entry
Domain event
Outbox event after publication
Subscription snapshot
License issuance record
```

## Soft delete

Do not physically delete commercial history.

Use:

```text
status = DELETED
```

or temporal validity.

---

# 64.1 Production-Grade Model Conventions (Normative)

The model snippets above are abbreviated for readability. To be production-grade, every model MUST satisfy the following when implemented. These rules are normative; where a snippet omits them, the rule still applies.

## Base class
- Extend `GlobalEntity` (catalog) or `TenantScopedEntity` (tenant-owned) — never raw `Base` with a hand-declared `id` (§4). This guarantees `id`, `created_at`/`created_by`, `updated_at`/`updated_by`, `version`, `metadata`.

## Money and quantities
- Every monetary or quantity column is `Numeric`, never float. Use an explicit precision/scale:

```python
amount: Mapped[Decimal] = mapped_column(Numeric(18, 6), nullable=False)
```

- Store either `Numeric(precision, scale)` or integer minor units. `precision` ≥ 18 for money; `scale` matched to the currency's `minor_unit` (rating/proration may need scale 6+). Never leave `Numeric` unbounded.

## Foreign keys (referential integrity)
- **Intra-platform references carry a real FK** (all live in the `license_*` schema): e.g. `price_id → license_price.id`, `plan_version_id → license_plan_version.id`, `subscription_id → license_subscription.id`, `meter_id → license_meter.id`.

```python
subscription_id: Mapped[uuid.UUID] = mapped_column(
    PGUUID(as_uuid=True),
    ForeignKey("license_subscription.id", ondelete="RESTRICT"),
    index=True,
)
```

- Use `ON DELETE RESTRICT` for commercial history; `CASCADE` only for owned child rows (e.g. tiers under a price) — and even then prefer soft delete (below).
- **Cross-context references carry NO FK**: `tenant_id`, `company_id`, `branch_id`, and user/customer references are owned by the core platform (§70). Index them instead.

## Soft delete and lifecycle
- This platform soft-deletes via `status = DELETED` (the `Status` enum), not an `is_deleted` boolean. Commercial history is never physically deleted (§64). Immutable records (§64) are never updated after finalization.
- Natural-key uniqueness must therefore be a **partial unique index excluding deleted rows**:

```python
Index("uq_license_product_code", "product_code", unique=True,
      postgresql_where=text("status <> 'DELETED'"))
```

## Uniqueness
- Global catalog codes (`product_code`, `feature_code`, `plan_code`, `coupon.code`, `signing_key.key_id`, `entitlement_token.jti`, …) are globally unique — as partial indexes per the rule above.
- Tenant-scoped document numbers are unique **within the tenant**, not globally:

```python
Index("uq_license_invoice_number", "tenant_id", "invoice_number", unique=True,
      postgresql_where=text("status <> 'DELETED'"))
```

  Apply to `invoice_number`, `quote_number`, `subscription_number`. (The inline `unique=True` shown on those columns above is shorthand for this composite, tenant-scoped index — replace it in implementation.)
- Idempotency/dedup keys are unique on their natural scope: `license_meter_event (tenant_id, meter_id, idempotency_key)`, `license_idempotency_key (tenant_id, key, operation)`, `license_inbox_event (source, event_id)`, `license_outbox_event` published-once semantics.

## Concurrency
- Mutable rows use the base integer `version` for optimistic locking (wired via `version_id_col`, §4). Do not add a second per-table `row_version`.

## Effective-dated rows
- Where `effective_from`/`effective_to` (or `valid_from`/`valid_until`) exist, enforce `CHECK (effective_to IS NULL OR effective_to >= effective_from)` and avoid overlapping active periods for the same scope unless the business rule allows it.

---

# 65. Critical Unique Constraints

Recommended constraints include:

```text
license_product.product_code
license_feature.feature_code
license_module.module_code
license_plan.plan_code
license_price.price_code
license_coupon.code
license_subscription.subscription_number
license_invoice.invoice_number
license_quote.quote_number
license_license_key.license_number
license_marketplace_app.app_code
license_marketplace_publisher.publisher_code
license_signing_key.key_id
license_entitlement_token.jti
```

Tenant-scoped numbers should use:

```text
tenant_id + subscription_number
tenant_id + invoice_number
tenant_id + quote_number
```

where global uniqueness is not required.

---

# 66. Critical Indexes

At minimum:

```text
subscription:
tenant_id + status
tenant_id + current_period_end
tenant_id + plan_version_id

subscription_item:
subscription_id + status

entitlement:
tenant_id + subscription_id
tenant_id + status
tenant_id + entitlement_version

usage:
tenant_id + subscription_id + meter_id
tenant_id + period_start + period_end

usage_event:
tenant_id + meter_id + occurred_at
tenant_id + idempotency_key

invoice:
tenant_id + status
tenant_id + subscription_id
tenant_id + due_date

outbox:
status + available_at

webhook_delivery:
status + next_retry_at

audit:
tenant_id + occurred_at
resource_type + resource_id
```

High-volume usage tables should eventually be partitioned by time and/or tenant.

---

# 67. Entitlement Compilation Pipeline

```text
Subscription Created/Changed
          │
          ▼
Transactional Outbox
          │
          ▼
SubscriptionChanged Event
          │
          ▼
Entitlement Worker
          │
          ├── Plan Version
          ├── Features
          ├── Add-ons
          ├── Contract
          ├── Overrides
          └── Promotions
          │
          ▼
Precedence Resolution
          │
          ▼
Final Entitlement
          │
          ▼
Hash Payload
          │
          ▼
Sign JWT
          │
          ▼
Persist Entitlement
          │
          ▼
Publish EntitlementChanged
```

---

# 68. Entitlement Precedence

Use deterministic precedence:

```text
1. Base Plan
2. Plan Version
3. Add-on
4. Contract
5. Promotional Grant
6. Administrative Override
7. Final Effective Entitlement
```

Every effective value should retain:

```text
source_type
source_id
priority
operation
```

so the result can be explained.

---

# 69. Subscription State Machine

```text
DRAFT
  │
  ├── TRIALING
  │      │
  │      └── ACTIVE
  │
  └── ACTIVE
          │
          ├── PAST_DUE
          │      │
          │      ├── ACTIVE
          │      └── SUSPENDED
          │
          ├── PAUSED
          │      └── ACTIVE
          │
          ├── CANCELED
          │
          └── EXPIRED
```

Do not allow arbitrary status changes. Every transition must be validated by a domain service.

---

# 70. Integration Boundaries

## Organization Platform (`p02_organization`)

```text
p02_organization
    ↓
tenant_id
company_id
branch_id
user/customer references
```

Licensing does not own those master records.

## Payment Platform

```text
p26_licensing
    │
    ├── PaymentRequested
    │
    ▼
payment
    │
    ├── PaymentSucceeded
    ├── PaymentFailed
    ├── RefundCompleted
    └── ChargebackOpened
```

## Tax Platform

```text
p26_licensing
    │
    │ TaxCalculationRequested
    ▼
tax
    │
    ▼
TaxCalculated
```

## Accounting Platform

```text
p26_licensing
    │
    │ InvoiceFinalized
    │ SubscriptionActivated
    │ PaymentSucceeded
    ▼
accounting
```

Accounting remains completely outside this database.

---

# 71. ERP Runtime Enforcement

ERP services should not query the licensing database on every request.

Recommended:

```text
p26_licensing
       │
       ▼
Signed Entitlement
       │
       ▼
IAM / Gateway
       │
       ▼
ERP Service
```

The ERP service receives:

```json
{
  "tenant_id": "...",
  "subscription_id": "...",
  "entitlement_version": 42,
  "features": {
    "finance": true,
    "hr": true
  },
  "limits": {
    "users": 100,
    "companies": 5
  }
}
```

---

# 72. Security Requirements

Required:

```text
JWT signing key rotation
KMS/HSM private-key storage
short-lived entitlement tokens
token revocation/versioning
idempotency
replay protection
audit logging
tenant isolation
row-level authorization
rate limiting
webhook signature verification
artifact signature verification
secret hashing
PII minimization
encryption at rest
TLS
```

Never store:

```text
raw payment card data
raw API secrets
private signing keys
raw hardware identifiers
```

---

# 73. Final Platform Boundary

The finished `p26_licensing` platform should look like:

```text
p26_licensing
│
├── Catalog
├── Products
├── Modules
├── Features
├── Plans
├── Plan Versions
├── Prices
├── Pricing Engine
├── Add-ons
│
├── Subscriptions
├── Subscription Items
├── Subscription Changes
├── Subscription Schedules
├── Trials
├── Contracts
│
├── Entitlements
├── Entitlement Compiler
├── Entitlement Overrides
├── Entitlement Tokens
├── Signing Keys
│
├── Metering
├── Usage
├── Usage Ledger
├── Quotas
├── Rate Limits
├── Usage Rating
│
├── Checkout
├── Quotes
├── Discounts
├── Coupons
├── Proration
│
├── Billing Cycle Coordination
├── Invoice Commercial Records
├── Payment References
├── Refund References
├── Chargeback References
├── Customer Credits
├── Dunning Lifecycle
│
├── Marketplace
├── Marketplace Apps
├── Marketplace Versions
├── Installations
├── Permissions
│
├── License Keys
├── Deployments
├── Activations
├── Hardware Binding
├── Revocation
│
├── Webhooks
├── Events
├── Outbox
├── Inbox
├── Idempotency
├── Notifications
├── Audit
├── Configuration
└── Reconciliation
```

This is the **pure subscription/licensing boundary**. Accounting and Finance are intentionally excluded.
