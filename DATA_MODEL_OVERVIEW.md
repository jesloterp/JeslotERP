# Data Model Overview

This is a **conceptual** map. It does not publish DDL, indexes, or internal
column lists.

## Naming

- PostgreSQL schema = domain name (`identity`, `org`, `bp`, `metadata`, …).
- Package number never appears in the schema name.
- Cross-schema references are UUIDs, not foreign keys.

## Foundation

| Schema | Owns |
|---|---|
| `identity` | Users, sessions, roles, permissions, MFA, OAuth clients, devices |
| `org` | Tenants, companies, branches, fiscal, UoM, FX, org masters |
| `configuration` | Definitions, scoped values, secret metadata, templates |
| `bp` | Partners, roles, satellites, KYC, match/merge, credit |
| `security` | Key metadata, policies, posture, WAF profiles |
| `sharing` | Teams, hierarchy, rules, grants |

## Content and workflow

| Schema | Owns |
|---|---|
| `metadata` | Dictionary, layouts, overlays, publish, FLS descriptors |
| `i18n` | Locales, messages, formats, TMS |
| `number_series` | Objects, definitions, counters, allocations |
| `media` | Objects, uploads, scan, attachments |
| `document` | DIR, versions, libraries, links, templates |
| `process` | Definitions, instances, tasks, SLA |
| `rules` | Definitions, tables, eval logs, compiled artifacts |
| `feature` | Flags, targeting, overrides, experiments |

## Async and observability

| Schema | Owns |
|---|---|
| `event_bus` | Types, topics, events, delivery, DLQ |
| `messaging` | Queues, jobs, leases, DLQ |
| `notification` | Templates, requests, deliveries, inbox |
| `cache` | Namespaces, policies, warmup |
| `scheduler` | Schedules, calendars, runs, tick locks |
| `search` | Indexes, mappings, saved searches, jobs |
| `audit` | Events, field changes, holds, exports |
| `logging` | Hot records, pipelines, sinks |
| `monitoring` | Metrics catalog, SLO, alerts |
| `api` | Specs, products, key hashes, limits |
| `integration` | Connectors, connections, mappings, pipelines, DLQ |

## Commercial, intelligence, lifecycle

| Schema | Owns |
|---|---|
| `reporting` | Datasets, reports, runs, exports |
| `dashboard` | Boards, widgets, bindings |
| `licensing` | Plans, subscriptions, entitlements, usage |
| `ai` | Deployments, assistants, corpora, guardrails |
| `extensibility` | Points, handlers, bindings, executions |
| `alm` | Environments, packages, promotions |
| `privacy` | Subjects, consent, holds, DSR |
| `output` | Templates, determinations, render jobs, spool |

## Business schemas (planned)

Registered names only — **NOT_FOUND** in the current implementation:

`finance`, `controlling`, `tax`, `treasury`, `sales`, `purchasing`,
`inventory`, `warehouse`, `logistics`, `manufacturing`, `quality`,
`maintenance`, `assets`, `projects`, `hcm`, `crm`, `service`.

## Integrity patterns (public)

- Tenant RLS on tenant-owned tables.
- Outbox rows in the same transaction as the write.
- Idempotency records on selected mutating APIs.
- Hash-chain integrity concepts on the audit plane.
- Published metadata / rules artifacts are versioned.

## What is not published

Physical table counts, migration revision ids, internal constraint names,
and production data.
