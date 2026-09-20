# API Architecture

## Surfaces (actual)

| Surface | Role |
|---|---|
| Public versioned HTTP | Operator and application clients |
| Internal HTTP | Service mesh / workers / verify |
| Module health | Loaded modules in dependency order |
| OpenAPI | Generated from the mounted application |

Exact hostnames are omitted.

## Conventions (public)

- JSON resource APIs grouped by package.
- Standard success / error envelope.
- Namespaced permissions on mutating routes.
- Idempotency on selected writes.
- Public versus internal prefixes so browser clients are not the worker plane.

## `p22_api` product plane

`p22_api` is the inbound **API product** catalog: specs, versions, products,
plans, hashed keys, rate policies, portal pages, bulk, and composite.

It is **not** a replacement for every package router. Packages still expose
their own `/api/v1/<area>` groups.

**Current:** hashed keys persist; optional distributed rate limiter;
bulk / composite exist.

**Not current:** OData, GraphQL, and a verified global edge deployment.

## `p23_integration`

Outbound connectors, mappings, pipelines, webhooks, and DLQ. Complementary
to inbound API products.

## Compatibility

Breaking metadata or API contract changes should go through publish /
versioning. Do not silently rename `pNN_*` packages.
