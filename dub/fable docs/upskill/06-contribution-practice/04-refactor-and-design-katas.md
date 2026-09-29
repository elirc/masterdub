# Refactor and Design Katas

Eight senior katas. These are *practice*, mostly not PRs — several would be rejected upstream as churn, which is itself part of the lesson. Do them on branches; the deliverable is the diff **plus a one-page write-up**. Self-grading criteria follow each.

## Kata 1: Fix the boundary leak — cache shape as a typed contract

Task: make the Redis-cached link shape (`RedisLinkProps` via `formatRedisLink`) a Zod schema; parse on cache read; delete the `as any` casts at [link.ts#L110-L115](../../../apps/web/lib/middleware/link.ts#L110-L115).
Self-grade — Solid: compiles, redirect works, one schema is the single source. Strong: you measured the parse cost on the hot path and either accepted it with numbers or used `schema.parse` only in dev/tests with a comment saying why; write-up names who owns the contract now.

## Kata 2: Design the outbox (paper only)

Task: full design doc for [project 1](03-senior-build-projects.md) *without* building it — table DDL, publisher pseudocode, ordering/dedup semantics, dual-write migration plan, and a one-paragraph steelman of NOT doing it.
Self-grade — Solid: covers atomicity, at-least-once, cleanup. Strong: the steelman is genuinely persuasive (TTL self-healing covers most effects; outbox adds a table, a cron, and a new failure mode) and your recommendation is scoped to webhooks only.

## Kata 3: Split a module — `withWorkspace` decomposition

Task: extract from [workspace.ts](../../../apps/web/lib/auth/workspace.ts) three pure(r) functions — `authenticateRequest` (credentials→session/token), `resolveWorkspaceAccess` (session+hint→workspace+role), `computePermissions` (role+scopes→permissions) — keeping `withWorkspace` as composition. No behavior change; unit tests for the third.
Self-grade — Solid: all integration behavior identical (how do you know? list your verification). Strong: the anonymous carve-out and machine-user promotion ended up *visible in the composition* rather than buried; write-up discusses what got harder (shared error/log context).

## Kata 4: Remove duplication — the six recordClick call sites

Task: the [ticket 7](01-good-first-tickets.md) refactor, executed, plus the write-up: why did the duplication exist (branch-local `finalUrl`), what did you couple by removing it, and where's the line (would you also unify the six `createResponseWithCookies` wrappings? why not?).
Self-grade — Strong: your helper has exactly one parameter that varies; you did NOT abstract the response-building (different rewrite/redirect/status shapes — premature unification), and you can say why in one sentence.

## Kata 5: Improve type safety — kill ten `any`s

Task: inventory every `any`/`as any`/`@ts-ignore` in [record-click.ts](../../../apps/web/lib/tinybird/record-click.ts) and [link.ts](../../../apps/web/lib/middleware/link.ts); fix the ten with the best safety-per-diff ratio.
Self-grade — Solid: `tsc --noEmit` clean, no runtime change. Strong: your write-up classifies the ones you *left* (boundary holes needing schemas — kata 1's job) vs fixed, showing you triage rather than crusade.

## Kata 6: Design a migration — split `Link.geo` JSON into a relation

Task: paper design to move geo targeting from a JSON column ([link.prisma#L34](../../../packages/prisma/schema/link.prisma#L34)) to a `LinkGeoTarget(linkId, country, url)` table. Expand-migrate-contract plan, backfill script shape, cached-shape implications, and the honest recommendation (probably: don't — JSON read by the hot path beats a join; the kata is realizing when the "cleaner" model is worse).
Self-grade — Strong: your plan keeps the redirect path join-free (denormalize back into the cache), and your recommendation section argues from access patterns, not aesthetics.

## Kata 7: Reduce a real N+1 — folder fetches in the links list

Task: investigate the per-card `useFolder` pattern ([link-card.tsx#L69-L72](../../../apps/web/ui/links/link-card.tsx#L69-L72)): with 100 links across 30 folders, how many `/api/folders/[id]` requests fire? (SWR dedupes identical ids.) Design the fix if warranted: `?includeFolder=true` on the links API vs a bulk `/api/folders?ids=` fetch vs current.
Self-grade — Solid: measured (network tab) before designing. Strong: your chosen fix accounts for folder ACLs (an include must apply the same permission filtering the folder endpoint does — the trap most candidates miss).

## Kata 8: Write the RFC — API versioning strategy

Task: the deprecation shims ([analytics route L27-L31](../../../apps/web/app/api/analytics/route.ts#L27-L31), [deprecated.ts](../../../apps/web/lib/zod/schemas/deprecated.ts)) are ad-hoc. Write the RFC for a consistent policy: usage telemetry per deprecated shape, sunset windows, changelog/SDK process, and how shims are marked in code. Use the RFC template from [07/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
Self-grade — Strong: your policy costs almost nothing for the common case (a comment + a metric), and you explicitly rejected URL-versioning (`/v2/`) with reasons specific to this repo's SDK-generation setup.
