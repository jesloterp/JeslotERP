# Development Guide

This guide is for architects and future contributors reading the **public**
specification. It does not describe how to obtain the private source.

## What you should understand first

1. [../MARKET_POSITION.md](../MARKET_POSITION.md) — why this is an independent product.
   Read [../TRADEMARKS.md](../TRADEMARKS.md) before using third-party names.
2. [../ARCHITECTURE.md](../ARCHITECTURE.md) — layers and plugin graph.
3. [../PLATFORM_PRINCIPLES.md](../PLATFORM_PRINCIPLES.md) — non-negotiables.
4. [../PACKAGES/P05_METADATA.md](../PACKAGES/P05_METADATA.md) and [../platforms/](../platforms/README.md).
5. [../BUSINESS_PLATFORM/HOW_TO_PLAN.md](../BUSINESS_PLATFORM/HOW_TO_PLAN.md)
   and [../BUSINESS_PLATFORM/REQUIREMENTS/](../BUSINESS_PLATFORM/REQUIREMENTS/README.md).

## Stack (public)

- Language: Python.
- HTTP: versioned JSON APIs.
- Persistence: PostgreSQL domain schemas + RLS.
- Shape: modular monolith, hexagonal packages.
- Client: a metadata-capable web desk is the intended operator UI.

## How a new platform capability should be added

1. Refuse a new `p34+` unless the registry is formally amended.
2. Prefer extending an existing owner (see principles table).
3. Ship: schema, module plugin, public/internal API, tests, GUIDE/SCHEMA/API.
4. Status moves Docs → Skeleton → SoR-Live. Never to Production from tests alone.

## How a new business module should be added

1. Take the next `bNN` from the business registry — do not reuse platform numbers.
2. Depend inward on platforms only.
3. Seed metadata, numbers, process, audit, sharing, output.
4. Follow [../BUSINESS_PLATFORM/BUSINESS_PLATFORM_TODO.md](../BUSINESS_PLATFORM/BUSINESS_PLATFORM_TODO.md).

## Testing expectations (conceptual)

- Contract tests for HTTP resource groups.
- Persistence tests when a database session is present.
- Provider tests must not invent live success without credentials.
