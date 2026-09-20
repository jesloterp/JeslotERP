# JeslotERP Organization Platform — Final Production Schema

**Version:** 1.3  
**Status:** **SoR-Live** — 28 original + 5 kept extras + 3 UoM/FX. Not Production.  
**Last reviewed:** 2026-09-12  
**Docs folder:** `docs/platforms/02_organization/`  
**Platform package:** `platforms.p02_organization`  
**PostgreSQL schema:** `org`

This document replaces the former organization schema reference file.  
`docs/` contains Markdown only. Runtime models live in `platforms/p02_organization/infrastructure/persistence/models/`.

---

## Schema reference (SQLAlchemy)

```python
from __future__ import annotations

import enum
import uuid
from datetime import date, datetime
from decimal import Decimal

from sqlalchemy import (
    Boolean, CheckConstraint, Date, DateTime, Enum as SAEnum,
    ForeignKey, ForeignKeyConstraint, Index, Integer, Numeric, String,
    Text, UniqueConstraint, func, text,
)
from sqlalchemy.dialects.postgresql import JSONB, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, declared_attr, mapped_column


# Partial-unique convention: business/natural keys (codes, numbers, names) are
# enforced with unique indexes scoped to WHERE is_deleted = false, so a
# soft-deleted row does not permanently reserve its code. Immutable-id hierarchy
# keys (used as composite-FK targets) stay as full UNIQUE constraints.
_ACTIVE_ONLY = text("is_deleted = false")


class Base(DeclarativeBase):
    pass


class RecordStatus(str, enum.Enum):
    ACTIVE = "ACTIVE"
    INACTIVE = "INACTIVE"
    SUSPENDED = "SUSPENDED"
    DELETED = "DELETED"


class TenantType(str, enum.Enum):
    STANDARD = "STANDARD"
    ENTERPRISE = "ENTERPRISE"
    PARTNER = "PARTNER"
    DEMO = "DEMO"


class DeploymentMode(str, enum.Enum):
    SHARED = "SHARED"
    DEDICATED = "DEDICATED"


class TaxRegistrationType(str, enum.Enum):
    REGULAR = "REGULAR"
    COMPOSITION = "COMPOSITION"
    SEZ = "SEZ"
    EXPORT = "EXPORT"
    ISD = "ISD"
    TDS = "TDS"
    TCS = "TCS"
    OTHER = "OTHER"


class TaxRegistrationStatus(str, enum.Enum):
    ACTIVE = "ACTIVE"
    INACTIVE = "INACTIVE"
    CANCELLED = "CANCELLED"


class FiscalPeriodStatus(str, enum.Enum):
    OPEN = "OPEN"
    SOFT_CLOSED = "SOFT_CLOSED"
    CLOSED = "CLOSED"


class UUIDMixin:
    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4,
        server_default=text("gen_random_uuid()")
    )


class AuditMixin:
    created_by: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=func.now())
    updated_by: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    updated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), onupdate=func.now())


class LifecycleMixin:
    status: Mapped[RecordStatus] = mapped_column(
        SAEnum(RecordStatus, name="record_status"), nullable=False,
        default=RecordStatus.ACTIVE, server_default=RecordStatus.ACTIVE.value
    )
    deleted_by: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    is_deleted: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))


class VersionMixin:
    version: Mapped[int] = mapped_column(Integer, nullable=False, default=1, server_default=text("1"))
    row_version: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False, default=uuid.uuid4, server_default=text("gen_random_uuid()"))


class ExtensionMixin:
    remarks: Mapped[str | None] = mapped_column(Text)
    metadata_fields: Mapped[dict | None] = mapped_column(JSONB)
    custom_fields: Mapped[dict | None] = mapped_column(JSONB)
    tags: Mapped[list | None] = mapped_column(JSONB)


class PlatformBase(Base, UUIDMixin, AuditMixin, LifecycleMixin, VersionMixin, ExtensionMixin):
    __abstract__ = True

    @declared_attr
    def __mapper_args__(cls):
        # Optimistic concurrency control: SQLAlchemy auto-increments `version` on
        # every UPDATE and raises StaleDataError when a concurrent write occurred.
        # (row_version UUID remains available as an external/audit change token.)
        return {"version_id_col": cls.version}


class TenantBase(PlatformBase):
    __abstract__ = True
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False, index=True)


# ============================================================
# GLOBAL MASTERS
# ============================================================

class OrgCountry(PlatformBase):
    __tablename__ = "org_country"
    iso2: Mapped[str] = mapped_column(String(2), nullable=False)
    iso3: Mapped[str] = mapped_column(String(3), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    phone_code: Mapped[str | None] = mapped_column(String(20))
    currency_code: Mapped[str | None] = mapped_column(String(10))
    nationality: Mapped[str | None] = mapped_column(String(100))
    continent: Mapped[str | None] = mapped_column(String(50))
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True, server_default=text("true"))
    __table_args__ = (
        Index("uq_org_country_iso2", "iso2", unique=True, postgresql_where=_ACTIVE_ONLY),
        Index("uq_org_country_iso3", "iso3", unique=True, postgresql_where=_ACTIVE_ONLY),
    )


class OrgState(PlatformBase):
    __tablename__ = "org_state"
    country_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("org_country.id", ondelete="RESTRICT"), nullable=False)
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    gst_code: Mapped[str | None] = mapped_column(String(20))
    capital: Mapped[str | None] = mapped_column(String(100))
    __table_args__ = (
        Index("uq_org_state_country_code", "country_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        Index("ix_org_state_country", "country_id"),
    )


class OrgCity(PlatformBase):
    __tablename__ = "org_city"
    country_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("org_country.id", ondelete="RESTRICT"), nullable=False)
    state_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("org_state.id", ondelete="RESTRICT"), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    postal_code: Mapped[str | None] = mapped_column(String(20))
    __table_args__ = (
        Index("uq_org_city_state_name_postal", "state_id", "name", "postal_code", unique=True, postgresql_where=_ACTIVE_ONLY),
    )


class OrgCurrency(PlatformBase):
    __tablename__ = "org_currency"
    code: Mapped[str] = mapped_column(String(10), nullable=False)
    iso_code: Mapped[str] = mapped_column(String(3), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    symbol: Mapped[str | None] = mapped_column(String(10))
    symbol_position: Mapped[str | None] = mapped_column(String(20))
    decimal_places: Mapped[int] = mapped_column(Integer, nullable=False, default=2, server_default=text("2"))
    rounding_precision: Mapped[int] = mapped_column(Integer, nullable=False, default=2, server_default=text("2"))
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True, server_default=text("true"))
    __table_args__ = (
        Index("uq_org_currency_code", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        Index("uq_org_currency_iso_code", "iso_code", unique=True, postgresql_where=_ACTIVE_ONLY),
        CheckConstraint("decimal_places >= 0", name="ck_currency_decimal_places"),
        CheckConstraint("rounding_precision >= 0", name="ck_currency_rounding_precision"),
    )


class OrgLanguage(PlatformBase):
    __tablename__ = "org_language"
    code: Mapped[str] = mapped_column(String(10), nullable=False)
    iso_code: Mapped[str] = mapped_column(String(3), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    native_name: Mapped[str | None] = mapped_column(String(100))
    locale: Mapped[str] = mapped_column(String(20), nullable=False)
    direction: Mapped[str] = mapped_column(String(10), nullable=False, default="LTR", server_default="LTR")
    is_default: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("uq_org_language_code", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        Index("uq_org_language_iso_code", "iso_code", unique=True, postgresql_where=_ACTIVE_ONLY),
    )


class OrgTimezone(PlatformBase):
    __tablename__ = "org_timezone"
    timezone_name: Mapped[str] = mapped_column(String(100), nullable=False)
    country_code: Mapped[str | None] = mapped_column(String(10))
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True, server_default=text("true"))
    __table_args__ = (
        Index("uq_org_timezone_name", "timezone_name", unique=True, postgresql_where=_ACTIVE_ONLY),
    )


# ============================================================
# TENANT
# ============================================================

class OrgTenant(PlatformBase):
    __tablename__ = "org_tenant"
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    display_name: Mapped[str | None] = mapped_column(String(255))
    legal_name: Mapped[str | None] = mapped_column(String(255))
    tenant_type: Mapped[TenantType] = mapped_column(SAEnum(TenantType, name="tenant_type"), nullable=False, default=TenantType.STANDARD, server_default="STANDARD")
    parent_tenant_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_tenant.id", ondelete="RESTRICT"))
    email: Mapped[str | None] = mapped_column(String(255))
    phone: Mapped[str | None] = mapped_column(String(50))
    website: Mapped[str | None] = mapped_column(String(255))
    logo_file_id: Mapped[str | None] = mapped_column(String(255))
    favicon_file_id: Mapped[str | None] = mapped_column(String(255))
    default_language_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_language.id", ondelete="RESTRICT"))
    default_currency_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_currency.id", ondelete="RESTRICT"))
    default_timezone_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_timezone.id", ondelete="RESTRICT"))
    country_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_country.id", ondelete="RESTRICT"))
    state_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_state.id", ondelete="RESTRICT"))
    city_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_city.id", ondelete="RESTRICT"))
    address_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    is_demo: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    trial_end_date: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    go_live_date: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    __table_args__ = (
        Index("uq_org_tenant_code", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        # A tenant's own address must belong to that tenant (org_tenant.id IS the tenant_id).
        ForeignKeyConstraint(["address_id", "id"], ["org_address.id", "org_address.tenant_id"], ondelete="SET NULL", name="fk_tenant_address_same_tenant"),
    )


class OrgTenantSetting(TenantBase):
    __tablename__ = "org_tenant_setting"
    key: Mapped[str] = mapped_column(String(100), nullable=False)
    value: Mapped[dict | None] = mapped_column(JSONB)
    is_encrypted: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("uq_org_tenant_setting", "tenant_id", "key", unique=True, postgresql_where=_ACTIVE_ONLY),
    )


class OrgTenantBranding(TenantBase):
    __tablename__ = "org_tenant_branding"
    is_white_label: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    brand_name: Mapped[str | None] = mapped_column(String(255))
    brand_logo_file_id: Mapped[str | None] = mapped_column(String(255))
    brand_favicon_file_id: Mapped[str | None] = mapped_column(String(255))
    brand_primary_color: Mapped[str | None] = mapped_column(String(50))
    brand_secondary_color: Mapped[str | None] = mapped_column(String(50))
    brand_font: Mapped[str | None] = mapped_column(String(100))
    brand_css: Mapped[str | None] = mapped_column(Text)
    brand_domain: Mapped[str | None] = mapped_column(String(255))
    support_email: Mapped[str | None] = mapped_column(String(255))
    support_phone: Mapped[str | None] = mapped_column(String(50))
    __table_args__ = (
        Index("uq_org_tenant_branding", "tenant_id", unique=True, postgresql_where=_ACTIVE_ONLY),
        Index("ix_org_tenant_brand_domain", "brand_domain"),
    )


class OrgTenantDeployment(TenantBase):
    __tablename__ = "org_tenant_deployment"
    deployment_mode: Mapped[DeploymentMode] = mapped_column(SAEnum(DeploymentMode, name="deployment_mode"), nullable=False, default=DeploymentMode.SHARED, server_default="SHARED")
    storage_limit_mb: Mapped[int | None] = mapped_column(Integer)
    storage_used_mb: Mapped[int] = mapped_column(Integer, nullable=False, default=0, server_default=text("0"))
    database_provider: Mapped[str | None] = mapped_column(String(50))
    database_host_ref: Mapped[str | None] = mapped_column(String(255))
    database_name_ref: Mapped[str | None] = mapped_column(String(255))
    database_schema: Mapped[str | None] = mapped_column(String(100))
    __table_args__ = (
        Index("uq_org_tenant_deployment", "tenant_id", unique=True, postgresql_where=_ACTIVE_ONLY),
        CheckConstraint("storage_limit_mb IS NULL OR storage_limit_mb >= 0", name="ck_storage_limit"),
        CheckConstraint("storage_used_mb >= 0", name="ck_storage_used"),
    )


class OrgTenantSubscription(TenantBase):
    __tablename__ = "org_tenant_subscription"
    external_subscription_id: Mapped[str | None] = mapped_column(String(100))
    plan_code: Mapped[str | None] = mapped_column(String(100))
    status: Mapped[str] = mapped_column(String(50), nullable=False)
    starts_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    ends_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    __table_args__ = (Index("ix_org_tenant_subscription_status", "tenant_id", "status"),)


# ============================================================
# ADDRESS / COMPANY / BRANCH
# ============================================================

class OrgAddress(TenantBase):
    __tablename__ = "org_address"
    address_type: Mapped[str] = mapped_column(String(50), nullable=False)
    address_line1: Mapped[str] = mapped_column(String(255), nullable=False)
    address_line2: Mapped[str | None] = mapped_column(String(255))
    landmark: Mapped[str | None] = mapped_column(String(150))
    country_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_country.id", ondelete="RESTRICT"))
    state_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_state.id", ondelete="RESTRICT"))
    city_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_city.id", ondelete="RESTRICT"))
    postal_code: Mapped[str | None] = mapped_column(String(20))
    latitude: Mapped[Decimal | None] = mapped_column(Numeric(9, 6))
    longitude: Mapped[Decimal | None] = mapped_column(Numeric(9, 6))
    is_primary: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        # Composite-FK target so tenant-scoped rows can only reference same-tenant addresses.
        UniqueConstraint("id", "tenant_id", name="uq_org_address_id_tenant"),
        CheckConstraint("latitude IS NULL OR latitude BETWEEN -90 AND 90", name="ck_address_latitude"),
        CheckConstraint("longitude IS NULL OR longitude BETWEEN -180 AND 180", name="ck_address_longitude"),
    )


class OrgCompany(TenantBase):
    __tablename__ = "org_company"
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    legal_name: Mapped[str | None] = mapped_column(String(255))
    company_type: Mapped[str | None] = mapped_column(String(50))
    registration_number: Mapped[str | None] = mapped_column(String(100))
    pan_number: Mapped[str | None] = mapped_column(String(20))
    tan_number: Mapped[str | None] = mapped_column(String(20))
    cin_number: Mapped[str | None] = mapped_column(String(50))
    msme_number: Mapped[str | None] = mapped_column(String(50))
    email: Mapped[str | None] = mapped_column(String(255))
    phone: Mapped[str | None] = mapped_column(String(50))
    website: Mapped[str | None] = mapped_column(String(255))
    logo_file_id: Mapped[str | None] = mapped_column(String(255))
    currency_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_currency.id", ondelete="RESTRICT"))
    language_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_language.id", ondelete="RESTRICT"))
    timezone_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_timezone.id", ondelete="RESTRICT"))
    country_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_country.id", ondelete="RESTRICT"))
    state_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_state.id", ondelete="RESTRICT"))
    city_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_city.id", ondelete="RESTRICT"))
    address_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    is_head_office: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    parent_company_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_company.id", ondelete="RESTRICT"))
    __table_args__ = (
        Index("uq_org_company_tenant_code", "tenant_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", name="uq_org_company_id_tenant"),
        ForeignKeyConstraint(["address_id", "tenant_id"], ["org_address.id", "org_address.tenant_id"], ondelete="SET NULL", name="fk_company_address_same_tenant"),
        Index("ix_org_company_tenant_status", "tenant_id", "status"),
    )


class OrgBranch(TenantBase):
    __tablename__ = "org_branch"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    branch_type: Mapped[str | None] = mapped_column(String(50))
    parent_branch_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    manager_employee_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    email: Mapped[str | None] = mapped_column(String(255))
    phone: Mapped[str | None] = mapped_column(String(50))
    address_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    country_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_country.id", ondelete="RESTRICT"))
    state_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_state.id", ondelete="RESTRICT"))
    city_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_city.id", ondelete="RESTRICT"))
    warehouse_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    cost_center_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    profit_center_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    is_head_branch: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("uq_org_branch_company_code", "tenant_id", "company_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", "company_id", name="uq_org_branch_id_hierarchy"),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT", name="fk_branch_company_tenant"),
        ForeignKeyConstraint(["parent_branch_id", "tenant_id", "company_id"], ["org_branch.id", "org_branch.tenant_id", "org_branch.company_id"], ondelete="RESTRICT", name="fk_branch_parent_same_company"),
        ForeignKeyConstraint(["address_id", "tenant_id"], ["org_address.id", "org_address.tenant_id"], ondelete="SET NULL", name="fk_branch_address_same_tenant"),
        Index("ix_org_branch_tenant_company", "tenant_id", "company_id"),
    )


# ============================================================
# PEOPLE / HR ORGANIZATION
# ============================================================

class OrgDepartment(TenantBase):
    __tablename__ = "org_department"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    parent_department_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    manager_employee_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    description: Mapped[str | None] = mapped_column(String(255))
    __table_args__ = (
        Index("uq_org_department_company_code", "tenant_id", "company_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", "company_id", name="uq_org_department_id_hierarchy"),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["parent_department_id", "tenant_id", "company_id"], ["org_department.id", "org_department.tenant_id", "org_department.company_id"], ondelete="RESTRICT"),
    )


class OrgDesignation(TenantBase):
    __tablename__ = "org_designation"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    level: Mapped[int | None] = mapped_column(Integer)
    parent_designation_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    description: Mapped[str | None] = mapped_column(String(255))
    __table_args__ = (
        Index("uq_org_designation_company_code", "tenant_id", "company_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", "company_id", name="uq_org_designation_id_hierarchy"),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["parent_designation_id", "tenant_id", "company_id"], ["org_designation.id", "org_designation.tenant_id", "org_designation.company_id"], ondelete="RESTRICT"),
        CheckConstraint("level IS NULL OR level >= 0", name="ck_org_designation_level"),
    )


class OrgCostCenter(TenantBase):
    __tablename__ = "org_cost_center"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    code: Mapped[str] = mapped_column(String(100), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    parent_cost_center_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    manager_employee_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    __table_args__ = (
        Index("uq_org_cost_center_company_code", "tenant_id", "company_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", "company_id", name="uq_org_cost_center_id_hierarchy"),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["parent_cost_center_id", "tenant_id", "company_id"], ["org_cost_center.id", "org_cost_center.tenant_id", "org_cost_center.company_id"], ondelete="RESTRICT"),
    )


class OrgProfitCenter(TenantBase):
    __tablename__ = "org_profit_center"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    code: Mapped[str] = mapped_column(String(100), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    parent_profit_center_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    manager_employee_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    __table_args__ = (
        Index("uq_org_profit_center_company_code", "tenant_id", "company_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", "company_id", name="uq_org_profit_center_id_hierarchy"),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["parent_profit_center_id", "tenant_id", "company_id"], ["org_profit_center.id", "org_profit_center.tenant_id", "org_profit_center.company_id"], ondelete="RESTRICT"),
    )


class OrgEmployee(TenantBase):
    __tablename__ = "org_employee"
    user_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), index=True)  # reference to IAM; no FK across bounded contexts
    employee_code: Mapped[str] = mapped_column(String(50), nullable=False)
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    branch_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    department_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    designation_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    reporting_manager_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    joining_date: Mapped[date | None] = mapped_column(Date)
    employee_type: Mapped[str | None] = mapped_column(String(50))
    work_email: Mapped[str | None] = mapped_column(String(255))
    work_phone: Mapped[str | None] = mapped_column(String(50))
    office_location_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    default_cost_center_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    default_profit_center_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True, server_default=text("true"))
    __table_args__ = (
        Index("uq_org_employee_company_code", "tenant_id", "company_id", "employee_code", unique=True, postgresql_where=_ACTIVE_ONLY),
        Index("uq_org_employee_tenant_user", "tenant_id", "user_id", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", "company_id", "branch_id", name="uq_org_employee_hierarchy"),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["branch_id", "tenant_id", "company_id"], ["org_branch.id", "org_branch.tenant_id", "org_branch.company_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["department_id", "tenant_id", "company_id"], ["org_department.id", "org_department.tenant_id", "org_department.company_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["designation_id", "tenant_id", "company_id"], ["org_designation.id", "org_designation.tenant_id", "org_designation.company_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["default_cost_center_id", "tenant_id", "company_id"], ["org_cost_center.id", "org_cost_center.tenant_id", "org_cost_center.company_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["default_profit_center_id", "tenant_id", "company_id"], ["org_profit_center.id", "org_profit_center.tenant_id", "org_profit_center.company_id"], ondelete="RESTRICT"),
        # NOTE: same-branch manager is a deliberate constraint; widen to (id,tenant,company)
        # if cross-branch / matrix reporting is required. office_location_id is intentionally
        # unconstrained (its target — office vs warehouse location — is app-defined).
        ForeignKeyConstraint(["reporting_manager_id", "tenant_id", "company_id", "branch_id"], ["org_employee.id", "org_employee.tenant_id", "org_employee.company_id", "org_employee.branch_id"], ondelete="RESTRICT"),
    )


# ============================================================
# WAREHOUSE
# ============================================================

class OrgWarehouse(TenantBase):
    __tablename__ = "org_warehouse"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    branch_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    warehouse_type: Mapped[str | None] = mapped_column(String(50))
    manager_employee_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    address_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    allow_negative_stock: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("uq_org_warehouse_company_code", "tenant_id", "company_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        UniqueConstraint("id", "tenant_id", "company_id", "branch_id", name="uq_org_warehouse_hierarchy"),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["branch_id", "tenant_id", "company_id"], ["org_branch.id", "org_branch.tenant_id", "org_branch.company_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["address_id", "tenant_id"], ["org_address.id", "org_address.tenant_id"], ondelete="SET NULL", name="fk_warehouse_address_same_tenant"),
    )


class OrgLocation(TenantBase):
    __tablename__ = "org_location"
    warehouse_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    parent_location_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    location_type: Mapped[str | None] = mapped_column(String(50))
    barcode: Mapped[str | None] = mapped_column(String(100))
    __table_args__ = (
        Index("uq_org_location_warehouse_code", "tenant_id", "warehouse_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        ForeignKeyConstraint(["warehouse_id", "tenant_id"], ["org_warehouse.id", "org_warehouse.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["parent_location_id", "tenant_id", "warehouse_id"], ["org_location.id", "org_location.tenant_id", "org_location.warehouse_id"], ondelete="RESTRICT"),
    )


# ============================================================
# FINANCIAL CALENDAR
# ============================================================

class OrgFiscalYear(TenantBase):
    __tablename__ = "org_fiscal_year"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    start_date: Mapped[date] = mapped_column(Date, nullable=False)
    end_date: Mapped[date] = mapped_column(Date, nullable=False)
    is_current: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    is_closed: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("uq_org_fiscal_year_company_code", "tenant_id", "company_id", "code", unique=True, postgresql_where=_ACTIVE_ONLY),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        CheckConstraint("end_date >= start_date", name="ck_org_fiscal_year_dates"),
    )


class OrgFinancialPeriod(TenantBase):
    __tablename__ = "org_financial_period"
    fiscal_year_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    period_no: Mapped[int] = mapped_column(Integer, nullable=False)
    period_name: Mapped[str] = mapped_column(String(100), nullable=False)
    start_date: Mapped[date] = mapped_column(Date, nullable=False)
    end_date: Mapped[date] = mapped_column(Date, nullable=False)
    period_type: Mapped[str] = mapped_column(String(50), nullable=False, default="MONTHLY", server_default="MONTHLY")
    status: Mapped[FiscalPeriodStatus] = mapped_column(SAEnum(FiscalPeriodStatus, name="fiscal_period_status"), nullable=False, default=FiscalPeriodStatus.OPEN, server_default="OPEN")
    __table_args__ = (
        Index("uq_org_financial_period_year_no", "tenant_id", "fiscal_year_id", "period_no", unique=True, postgresql_where=_ACTIVE_ONLY),
        ForeignKeyConstraint(["fiscal_year_id", "tenant_id"], ["org_fiscal_year.id", "org_fiscal_year.tenant_id"], ondelete="RESTRICT"),
        CheckConstraint("period_no > 0", name="ck_org_financial_period_no"),
        CheckConstraint("end_date >= start_date", name="ck_org_financial_period_dates"),
    )


class OrgFiscalPeriodClosure(TenantBase):
    __tablename__ = "org_fiscal_period_closure"
    fiscal_period_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    branch_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    closed_by: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    closed_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    reopened_by: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    reopened_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    reason: Mapped[str | None] = mapped_column(Text)
    __table_args__ = (
        Index("uq_org_period_closure_scope", "tenant_id", "fiscal_period_id", "branch_id", unique=True, postgresql_where=_ACTIVE_ONLY),
        ForeignKeyConstraint(["fiscal_period_id", "tenant_id"], ["org_financial_period.id", "org_financial_period.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["branch_id", "tenant_id"], ["org_branch.id", "org_branch.tenant_id"], ondelete="RESTRICT"),
    )


class OrgWorkingCalendar(TenantBase):
    __tablename__ = "org_working_calendar"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    week_start: Mapped[int] = mapped_column(Integer, nullable=False, default=1, server_default=text("1"))
    working_days: Mapped[list] = mapped_column(JSONB, nullable=False)
    working_hours: Mapped[dict] = mapped_column(JSONB, nullable=False)
    timezone_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("org_timezone.id", ondelete="RESTRICT"), nullable=False)
    __table_args__ = (
        Index("uq_org_calendar_company_name", "tenant_id", "company_id", "name", unique=True, postgresql_where=_ACTIVE_ONLY),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        CheckConstraint("week_start BETWEEN 0 AND 6", name="ck_org_calendar_week_start"),
    )


class OrgHoliday(TenantBase):
    __tablename__ = "org_holiday"
    calendar_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    holiday_date: Mapped[date] = mapped_column(Date, nullable=False)
    holiday_name: Mapped[str] = mapped_column(String(100), nullable=False)
    holiday_type: Mapped[str | None] = mapped_column(String(50))
    is_optional: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("uq_org_holiday_calendar_date", "tenant_id", "calendar_id", "holiday_date", unique=True, postgresql_where=_ACTIVE_ONLY),
        ForeignKeyConstraint(["calendar_id", "tenant_id"], ["org_working_calendar.id", "org_working_calendar.tenant_id"], ondelete="RESTRICT"),
    )


# ============================================================
# TAX / CONTACT
# ============================================================

class OrgTaxRegistration(TenantBase):
    __tablename__ = "org_tax_registration"
    company_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    branch_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True))
    registration_type: Mapped[TaxRegistrationType] = mapped_column(SAEnum(TaxRegistrationType, name="tax_registration_type"), nullable=False)
    registration_number: Mapped[str] = mapped_column(String(100), nullable=False)
    state_id: Mapped[uuid.UUID | None] = mapped_column(UUID(as_uuid=True), ForeignKey("org_state.id", ondelete="RESTRICT"))
    status: Mapped[TaxRegistrationStatus] = mapped_column(SAEnum(TaxRegistrationStatus, name="tax_registration_status"), nullable=False, default=TaxRegistrationStatus.ACTIVE, server_default="ACTIVE")
    effective_from: Mapped[date | None] = mapped_column(Date)
    effective_to: Mapped[date | None] = mapped_column(Date)
    is_primary: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("uq_org_tax_registration_number", "tenant_id", "registration_number", unique=True, postgresql_where=_ACTIVE_ONLY),
        ForeignKeyConstraint(["company_id", "tenant_id"], ["org_company.id", "org_company.tenant_id"], ondelete="RESTRICT"),
        ForeignKeyConstraint(["branch_id", "tenant_id", "company_id"], ["org_branch.id", "org_branch.tenant_id", "org_branch.company_id"], ondelete="RESTRICT"),
        CheckConstraint("effective_to IS NULL OR effective_from IS NULL OR effective_to >= effective_from", name="ck_org_tax_registration_dates"),
    )


class OrgContact(TenantBase):
    __tablename__ = "org_contact"
    # Polymorphic owner (COMPANY / BRANCH / WAREHOUSE / TENANT / ...). No cross-table
    # FK by design; integrity of (owner_type, owner_id) is enforced at the service layer.
    owner_type: Mapped[str] = mapped_column(String(50), nullable=False)
    owner_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    contact_type: Mapped[str] = mapped_column(String(50), nullable=False)
    person_name: Mapped[str] = mapped_column(String(150), nullable=False)
    designation: Mapped[str | None] = mapped_column(String(100))
    email: Mapped[str | None] = mapped_column(String(255))
    phone: Mapped[str | None] = mapped_column(String(50))
    mobile: Mapped[str | None] = mapped_column(String(50))
    is_primary: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, server_default=text("false"))
    __table_args__ = (
        Index("ix_org_contact_owner", "tenant_id", "owner_type", "owner_id"),
        # At most one primary contact per owner (active rows only).
        Index("uq_org_contact_primary", "tenant_id", "owner_type", "owner_id", unique=True,
              postgresql_where=text("is_primary = true AND is_deleted = false")),
    )


# ============================================================
# NOTES FOR MIGRATIONS / SECURITY
# ============================================================
# 1. Enable pgcrypto if using gen_random_uuid(): CREATE EXTENSION IF NOT EXISTS pgcrypto;
# 2. Keep IAM as owner of users/roles/permissions. org_employee.user_id is a bounded-context reference.
# 3. Enforce tenant context at application/service layer and preferably PostgreSQL RLS for defense-in-depth.
# 4. Never store plaintext DB passwords or full connection strings in org_tenant/org_tenant_deployment.
# 5. Add Alembic migrations; do not call Base.metadata.create_all() in production.
# 6. Natural-key uniqueness (codes/numbers/names) now uses partial unique indexes
#    scoped to WHERE is_deleted = false, so soft-deleted rows release their key.
#    Immutable-id hierarchy keys (composite-FK targets) remain full UNIQUE constraints.
# 7. Still enforce remaining "one current/primary/default" rules in migrations where a
#    scope isn't obvious, e.g. org_fiscal_year.is_current per company,
#    org_tax_registration.is_primary per company, org_company.is_head_office per tenant,
#    org_branch.is_head_branch per company, org_language.is_default (global).
#    org_contact primary-per-owner is already enforced here.
# 8. Optimistic concurrency uses PlatformBase.version (version_id_col); bump requires the
#    ORM UPDATE path — bulk/raw SQL writes must maintain `version` themselves.
# 9. org_address is tenant-scoped (TenantBase); all address_id references are composite
#    (address_id, tenant_id) FKs so an address cannot cross tenants. If truly global/shared
#    addresses are needed, model them as a separate table rather than relaxing this.
# 10. org_employee.reporting_manager_id is same-branch by FK; widen the composite FK if
#     cross-branch / matrix reporting is a requirement.
```

## Extra ORM tables (AUD-016 / HYG-016 — kept)

The 28 classes above are the original schema reference. Runtime also ships these five **kept** tables. **Document, do not migrate back.** Do not invent p34.

They live in `platforms/p02_organization` and Alembic `org`. Mixins (`id`, audit, lifecycle, `tenant_id` where TenantBase) are the same as the 28.

### `org_bank_account` (tenant + RLS)

Satellite bank account for a company (or other owner). Seed and company owner use this table.

| Column | Type | Notes |
|---|---|---|
| `owner_type` | varchar(50) | default `COMPANY` |
| `owner_id` | uuid | nullable |
| `bank_name` | varchar(255) | required |
| `account_name` | varchar(255) | |
| `account_number` | varchar(100) | partial unique per tenant |
| `ifsc_code` | varchar(50) | |
| `swift_code` | varchar(50) | |
| `branch_name` | varchar(255) | |
| `account_type` | varchar(50) | |
| `currency_id` | uuid | FK `org.org_currency.id` (same schema) |
| `is_primary` | bool | |
| `iban` | varchar(50) | |

Index `uq_org_bank_account_number` on (`tenant_id`, `account_number`) where not deleted. Index `ix_org_bank_account_owner`.

### `org_config_system_code`

SuperAdmin / seed system codes. `tenant_id` may be null for platform-wide rows.

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | uuid | nullable (override TenantBase) |
| `module` | varchar(100) | default `ORGANIZATION` |
| `code` | varchar(100) | required |
| `name` | varchar(255) | required |
| `description` | text | |
| `sort_order` | int | default 0 |
| `is_system` | bool | seed-owned when true |

Partial unique `uq_org_config_system_code` on (`tenant_id`, `company_id`, `module`, `code`).

### `org_onboarding_session` (platform)

SuperAdmin onboarding draft. Not tenant-RLS.

| Column | Type | Notes |
|---|---|---|
| `tenant_name` | varchar(255) | required |
| `step` | int | default 1 |
| `payload` | jsonb | draft fields |
| `error_log` | text | |
| `result_tenant_id` | uuid | set on finish |

### `org_outbox_event`

Platform outbox plumbing for organization domain events.

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | uuid | nullable |
| `aggregate_type` | varchar(100) | |
| `aggregate_id` | varchar(255) | |
| `event_type` | varchar(150) | required |
| `payload` | jsonb | required |
| `status` | varchar(30) | default `PENDING` |
| `attempts` | int | default 0 |
| `available_at` | timestamptz | |
| `processed_at` | timestamptz | |
| `last_error` | text | |
| `created_at` | timestamptz | |
| `is_deleted` | bool | |

Indexes `ix_org_outbox_status_available`, `ix_org_outbox_event_type`.

### `org_idempotency_key`

HTTP idempotency plumbing for retry-sensitive ORG POSTs.

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | uuid | nullable |
| `actor_id` | uuid | |
| `idempotency_key` | varchar(255) | required |
| `route` | varchar(255) | required |
| `request_hash` | varchar(64) | SHA-256 of body |
| `response_status` | int | default 200 |
| `response_body` | jsonb | replay payload |
| `created_at` | timestamptz | |

Unique `uq_org_idempotency_scope` on (`tenant_id`, `actor_id`, `route`, `idempotency_key`).

`OrgRegistration` is an alias of `org_tax_registration`. Fiscal `FiscalYear` / `FiscalPeriod` / `FiscalPeriodClosure` are aliases of the documented fiscal tables.

**Runtime inventory:** 36 `__tablename__` values (28 original + these 5 + 3 UoM/FX). A new ORM table must be added here before it ships.

## UoM + FX (TASK-SOR-027 — stay in p02)

Cache is never the source of these rates. Empty HTTP catalog is `[]`.

| Table | Scope | Purpose |
|---|---|---|
| `org_uom` | Platform | Unit catalog (`uom_code`, `dimension`, `is_base`) |
| `org_uom_conversion` | Platform | Directed factor: amount × numerator / denominator (inverse allowed) |
| `org_fx_rate` | Tenant + RLS | FX rate (`from_currency`, `to_currency`, `rate_type`, `valid_from`) |

Alembic: `f02c1d2e3f40` after `f33b1c2d3e4f`.
