# Localization

**Package:** `p06_localization`  
**Status:** IMPLEMENTED (kernel)

## Current capability

- Locale catalog and effective ICU message resolve.
- Overrides, format profiles, glossary, translation memory.
- Translation management tasks, coverage, import / export, packs.
- Seeded locales used by the metadata partner pilot (English plus additional
  Indic locales).
- Notification and output consume locale conceptually.

## Does not own

- Metadata field definitions (stores `label_key` only).
- Organization structure.
- Secret values for machine-translation providers.

## Planned

- Machine translation as a silent publish path is **rejected** by policy.
- Full UI translation of every platform console is ongoing, not claimed complete.
