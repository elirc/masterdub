# 01 — Codebase Cartography

Before you can change a system safely you must know where things live, what owns what, and how requests actually move. This module builds the map.

Read in order:

1. [01-system-map.md](01-system-map.md) — monorepo shape, runtime surfaces, ownership.
2. [02-file-reading-order.md](02-file-reading-order.md) — 28 files in the order that builds understanding fastest, with junior/mid/senior paths.
3. [03-domain-glossary.md](03-domain-glossary.md) — the product nouns (workspace vs project, link vs shortLink, program vs partner) and where each lives in code.
4. [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) — pnpm/turbo/Next, env boundaries, where code executes.
5. [05-key-flows.md](05-key-flows.md) — seven end-to-end traces. **The most important file in this module**; everything else in the curriculum cross-references it.

Transferable skill: the mapping *method* — hostnames → middlewares → wrappers → domain functions → stores — works on any web codebase. Interviewers ask "how do you ramp up on a large codebase?"; your answer after this module is a concrete procedure, not "I read the docs."
