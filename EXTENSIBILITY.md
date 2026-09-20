# Extensibility

## Current capability

`p29_extensibility` is an allow-listed hook pipeline:

- Declared extension points.
- Handlers that are allow-listed before bind.
- Isolated execution with timeout.
- Execution history.
- Kernel invoke façade used by partner save, metadata publish, and process lifecycle.

Stored definitions are **not** evaluated as Python.

`p05_metadata` additionally allows tenant / pack overlays and custom fields
without deleting system fields.

`p30_alm` moves sealed packages across landscapes (deploy agents fail closed
without endpoints).

## Target capability

- Hook coverage on every transactional business aggregate.
- Industry packs (tax localization, vertical documents) as ALM artifacts.
- Connector adapters on `p23_integration` as the external extension model.
- Renderer and channel adapters on `p32_output` / `p15_notification`.

## What is forbidden

- `eval` / arbitrary code from the database.
- Business modules importing platform ORM.
- Platform packages depending on `business/bNN_*`.
- Silent schema mutation from metadata drift detection.

## How a future contributor should extend

1. Prefer metadata overlays and layouts.
2. Prefer rules and process before new Python.
3. If Python is required, register an allow-listed handler.
4. Ship content as a sealed package, not a fork.

See [DEVELOPMENT/EXTENSION_GUIDE.md](DEVELOPMENT/EXTENSION_GUIDE.md).
