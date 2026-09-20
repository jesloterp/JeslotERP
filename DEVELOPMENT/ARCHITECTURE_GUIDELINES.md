# Architecture Guidelines

1. **Canonical names.** Always `p05_metadata`, never “the metadata service”
   as an identifier.
2. **Hexagonal packages.** Domain and application do not import adapters.
3. **No ORM imports across packages.** Gateways, HTTP, events.
4. **No cross-schema FKs.**
5. **Namespaced permissions.**
6. **Outbox in the same transaction.**
7. **Fail closed.**
8. **Safe AST / allow-listed hooks.**
9. **Business downward only.**
10. **Status honesty.** SoR-Live ≠ Production.
11. **One owner.** If two packages seem to own it, merge or write a boundary.
12. **UoM/FX stay in `p02_organization`.**
13. **Bytes vs document vs print** stay in `p08` / `p09` / `p32`.
14. **Facts vs jobs vs notifications** stay in `p13` / `p14` / `p15`.
