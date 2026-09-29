# 03 — Architecture and Patterns

The judgment module. Everything here trains the mid-level question ("is this the right pattern?") and the senior question ("what does this commit us to?").

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — the layer cake, what each owns, and where it leaks.
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — schema, indexes, transactions, how to change it safely.
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — every validation layer; authn vs authz; tenant isolation.
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — the side-effect map; idempotency, retries, failure visibility.
5. [05-pattern-catalog.md](05-pattern-catalog.md) — 16 pattern cards for recognition.
6. [06-architecture-critique.md](06-architecture-critique.md) — strengths, risks, and what I'd change owning this for 3 months. Doubles as system-design interview material ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)).

Vocabulary this module makes yours (define each in context as you go — they're interview words): invariant, boundary, contract, ownership, idempotency, isolation, consistency, blast radius, observability.
