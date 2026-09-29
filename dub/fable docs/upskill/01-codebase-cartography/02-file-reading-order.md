# File Reading Order

28 files, ordered so each one makes the next legible. Per file: why it matters, what to look for, what to ignore. Junior path = 1–12. Mid path adds 13–22. Senior path adds 23–28 and reads everything with the critique lens ([03/06](../03-architecture-and-patterns/06-architecture-critique.md)).

## Junior path (locate & explain)

1. [package.json](../../../package.json) — scripts tell you the verbs. Ignore: devDependency versions.
2. [pnpm-workspace.yaml](../../../pnpm-workspace.yaml) + [turbo.json](../../../turbo.json) — workspace globs and build DAG (`dependsOn: ["^build"]`). Ignore: cache output globs.
3. [apps/web/package.json](../../../apps/web/package.json) — the app's verbs (dev/test/e2e/openapi). Look for: `test` implies prisma generate; `dev` runs two processes.
4. [apps/web/middleware.ts](../../../apps/web/middleware.ts) — hostname dispatch. Look for: the matcher regex (what *doesn't* go through middleware).
5. [apps/web/lib/middleware/utils/parse.ts](../../../apps/web/lib/middleware/utils/parse.ts) — how domain/key/fullPath are derived. Everything downstream trusts this.
6. [apps/web/lib/middleware/link.ts](../../../apps/web/lib/middleware/link.ts) — the product in one file. Look for: guard order; every `ev.waitUntil`. Ignore on first pass: deep-link/AppsFlyer branches.
7. [apps/web/lib/api/links/cache.ts](../../../apps/web/lib/api/links/cache.ts) — tiers + TTLs. Look for: which methods invalidate what.
8. [apps/web/lib/auth/workspace.ts](../../../apps/web/lib/auth/workspace.ts) — the auth spine. Slow read. Look for: the order of gates.
9. [apps/web/app/api/links/route.ts](../../../apps/web/app/api/links/route.ts) — the canonical route. Look for: how little it does itself.
10. [apps/web/lib/api/links/process-link.ts](../../../apps/web/lib/api/links/process-link.ts) — validation as data. Look for: the return-type union; plan gates.
11. [apps/web/lib/api/links/create-link.ts](../../../apps/web/lib/api/links/create-link.ts) — the write + fan-out. Look for: what's inside vs outside the DB write.
12. [packages/prisma/schema/link.prisma](../../../packages/prisma/schema/link.prisma) + [workspace.prisma](../../../packages/prisma/schema/workspace.prisma) — the data heart. Look for: index comments (each names its query); `Project` = workspace.

## Mid path (modify safely)

13. [apps/web/lib/api/errors.ts](../../../apps/web/lib/api/errors.ts) — the error contract. Look for: what happens to an error type you've never seen.
14. [apps/web/lib/tinybird/record-click.ts](../../../apps/web/lib/tinybird/record-click.ts) — async reliability in miniature. Look for: every silent-skip path.
15. [apps/web/lib/zod/schemas/links.ts](../../../apps/web/lib/zod/schemas/links.ts) — request/response contracts. Look for: schema variants (create vs update vs query) and shared fragments.
16. [apps/web/lib/webhook/qstash.ts](../../../apps/web/lib/webhook/qstash.ts) + [lib/webhook/failure.ts](../../../apps/web/lib/webhook/failure.ts) — outbound delivery. Look for: signature, callbacks, disable policy.
17. [apps/web/lib/cron/verify-qstash.ts](../../../apps/web/lib/cron/verify-qstash.ts) + one job under [app/(ee)/api/cron/links/](../../../apps/web/app/(ee)/api/cron/links/) — the inbound background surface.
18. [apps/web/lib/swr/use-links.ts](../../../apps/web/lib/swr/use-links.ts) + [lib/swr/use-workspace.ts](../../../apps/web/lib/swr/use-workspace.ts) — client data layer. Look for: key construction.
19. [apps/web/ui/links/link-card.tsx](../../../apps/web/ui/links/link-card.tsx) — a production component: memo, context, conditional SWR (`enabled`), prefetch-on-visible ([#L79-L84](../../../apps/web/ui/links/link-card.tsx#L79-L84)).
20. [apps/web/lib/actions/safe-action.ts](../../../apps/web/lib/actions/safe-action.ts) — the *other* mutation path (server actions). Look for: how its auth duplicates withWorkspace's ideas.
21. [apps/web/tests/utils/integration.ts](../../../apps/web/tests/utils/integration.ts) + [tests/links/create-link.test.ts](../../../apps/web/tests/links/create-link.test.ts) — the testing model. Look for: what these tests can and cannot catch.
22. [apps/web/lib/api/rbac/permissions.ts](../../../apps/web/lib/api/rbac/permissions.ts) + [lib/api/tokens/scopes.ts](../../../apps/web/lib/api/tokens/scopes.ts) — the permission vocabulary.

## Senior path (critique & design)

23. [apps/web/lib/planetscale/](../../../apps/web/lib/planetscale/) — why an HTTP SQL driver exists next to Prisma.
24. [apps/web/lib/upstash/](../../../apps/web/lib/upstash/) incl. `redis-streams/` — Redis as cache, limiter, stream bus.
25. [apps/web/lib/api/links/bulk-create-links.ts](../../../apps/web/lib/api/links/bulk-create-links.ts) — how the single-item patterns scale to batches.
26. [apps/web/lib/folder/permissions.ts](../../../apps/web/lib/folder/permissions.ts) — the second authorization dimension (intra-workspace ACLs).
27. [apps/web/instrumentation.ts](../../../apps/web/instrumentation.ts) + [next.config.js](../../../apps/web/next.config.js) — boot-time hooks, build config, rewrites.
28. [apps/web/app/(ee)/api/cron/usage/](../../../apps/web/app/(ee)/api/cron/usage/) — billing/usage reconciliation; where the async counters finally meet money.

Pause-and-predict discipline: before opening each file, write one sentence predicting what it does; score yourself after. Consistently wrong predictions in an area = that's where your next deep-dive goes.
