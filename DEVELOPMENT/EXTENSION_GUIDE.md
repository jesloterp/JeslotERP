# Extension Guide

## Preferred order

1. **Metadata overlay** — extra fields, layouts, validation.
2. **Configuration value** — policy numbers and toggles.
3. **Feature flag** — rollout.
4. **Rule** — decisions.
5. **Process** — human/system steps.
6. **Allow-listed hook** — Python that cannot live in data.
7. **Integration connector** — external system.
8. **Industry pack via ALM** — portable content.
9. **New business module** — new ledger or document family.
10. **New platform package** — last resort; registry amendment required.

## Forbidden

- Evaluating stored source text.
- Forking kernel packages per customer.
- Embedding secrets in packs.
- Platform depending on a customer extension.

See [../EXTENSIBILITY.md](../EXTENSIBILITY.md) and [../PACKAGES/P29_EXTENSIBILITY.md](../PACKAGES/P29_EXTENSIBILITY.md).
