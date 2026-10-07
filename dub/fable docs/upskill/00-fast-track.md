# 00 — Fast Track: One Weekend in Dub

Goal: by Sunday night you can (a) explain how a short link resolves, (b) explain how a link gets created and cached, (c) run the test suite's shape in your head, and (d) narrate one flow aloud as if in an interview.

## Saturday morning — install and orient (2–3 h)

All commands run from the repo root (`dub/`) unless noted. Every command here is __inferred__ from [package.json](../../package.json) and [apps/web/package.json](../../apps/web/package.json) — they were not executed during authoring because the app needs external services (PlanetScale MySQL, Upstash Redis, Tinybird) and a populated `.env`.

```bash
pnpm install                 # inferred — pnpm@9.15.9 per packageManager field
pnpm build:packages          # inferred — builds packages/* so apps/web can import them
cd apps/web
cp .env.example .env         # inferred — check apps/web for the example file; fill in at minimum DATABASE_URL, UPSTASH_*, NEXTAUTH_*
pnpm dev                     # inferred — Next.js on http://localhost:8888 + Prisma Studio, via concurrently
```

Reality check: Dub is a SaaS with heavy infra dependencies. **Do not stall your weekend on a full local boot.** There is a [docker-compose.yml](../../apps/web/docker-compose.yml) in `apps/web` for local backing services — investigate it if you want a real boot. Otherwise, everything below works by *reading*, which is the actual skill being trained.

Note the local-dev escape hatches you'll keep seeing: geolocation is faked off-Vercel ([lib/middleware/link.ts#L315-L318](../../apps/web/lib/middleware/link.ts#L315-L318)), and QStash signature verification is skipped when not on Vercel ([lib/cron/verify-qstash.ts#L18-L21](../../apps/web/lib/cron/verify-qstash.ts#L18-L21)).

## Saturday afternoon — the first 10 files, in order

Open these in order. For each: read top-to-bottom once, then answer the pause-question *before* moving on.

1. [package.json](../../package.json) + [pnpm-workspace.yaml](../../pnpm-workspace.yaml) — monorepo shape, publishable packages. *Q: which packages are published to npm?* (Look at the `publish-*` scripts.)
2. [apps/web/middleware.ts](../../apps/web/middleware.ts) — the front door. Hostname-based dispatch. *Q: what happens to a request for `dub.sh/github`? For `app.dub.co/settings`?*
3. [apps/web/lib/middleware/link.ts](../../apps/web/lib/middleware/link.ts) — the money path: cache lookup, guard branches, redirect. *Q: in what order are password / banned / expired checked, and why does order matter?*
4. [apps/web/lib/api/links/cache.ts](../../apps/web/lib/api/links/cache.ts) — three cache tiers. *Q: why does an in-process LRU with a 5-second TTL exist in front of Redis?*
5. [apps/web/lib/auth/workspace.ts](../../apps/web/lib/auth/workspace.ts) — the authorization spine. Read it slowly; it's the most interview-dense file in the repo. *Q: what are the two authentication modes, and where does tenant isolation actually happen?*
6. [apps/web/app/api/links/route.ts](../../apps/web/app/api/links/route.ts) — a complete REST endpoint in ~110 lines. *Q: why does POST call `processLink` and `createLink` as two separate steps?*
7. [apps/web/lib/api/links/create-link.ts](../../apps/web/lib/api/links/create-link.ts) — write path + async fan-out. *Q: list everything that happens inside `waitUntil` and what breaks if each fails.*
8. [apps/web/lib/tinybird/record-click.ts](../../apps/web/lib/tinybird/record-click.ts) — the analytics write path. *Q: why `Promise.allSettled` and not `Promise.all`?*
9. [packages/prisma/schema/link.prisma](../../packages/prisma/schema/link.prisma) — the central entity. *Q: why do `clicks`/`leads`/`sales` counters live on the row when Tinybird stores every event?*
10. [apps/web/tests/links/create-link.test.ts](../../apps/web/tests/links/create-link.test.ts) + [tests/utils/integration.ts](../../apps/web/tests/utils/integration.ts) — how this repo tests. *Q: are these unit tests? What do they need to run?* (Answer: no — they hit a live deployment with a real token.)

**Check your Saturday answers** (each is verifiable by reading the cited file):

- Q1: the root `package.json` has seven `publish-*` scripts — `@dub/cli`, `@dub/embed-core`, `@dub/embed-react`, `@dub/prisma`, `@dub/tailwind-config`, `@dub/ui`, `@dub/utils`. `apps/web` is not published.
- Q3: in `lib/middleware/link.ts` the order is password (L188), banned — the `LEGAL_WORKSPACE_ID` check (L213), disabled (L224), expired (L234). If you put "expired" first, an expired *banned* link would show the expired page instead of the banned one.
- Q4: the LRU is configured at `lib/api/links/cache.ts` L19-L20 (`max: 10000`, `ttl: 5000`). It absorbs bursts of clicks on one hot link inside a single instance before they reach Redis.
- Q5: `withWorkspace` reads an `Authorization: Bearer …` API key (L105-L111) and otherwise falls back to the session from `getSession`; tenant isolation is the workspace lookup that both modes feed, not the UI.

## Sunday — two end-to-end traces

### Trace 1: `GET https://dub.sh/github` (read path)

[middleware.ts#L35-L90](../../apps/web/middleware.ts#L35-L90) dispatch → [lib/middleware/link.ts#L48-L63](../../apps/web/lib/middleware/link.ts#L48-L63) key normalization → [#L85-L132](../../apps/web/lib/middleware/link.ts#L85-L132) cache-then-DB lookup → guard branches (password L188, banned L213, disabled L224, expired L234) → [#L547-L579](../../apps/web/lib/middleware/link.ts#L547-L579) redirect + `ev.waitUntil(recordClick)` → [lib/tinybird/record-click.ts#L177-L230](../../apps/web/lib/tinybird/record-click.ts#L177-L230) async fan-out.

Fill in a trace table (step / file / what happens / what can fail). Compare against [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md) Flow 1.

### Trace 2: `POST /api/links` (write path)

[app/api/links/route.ts#L48-L110](../../apps/web/app/api/links/route.ts#L48-L110) → `withWorkspace` ([lib/auth/workspace.ts#L58-L80](../../apps/web/lib/auth/workspace.ts#L58-L80)) → Zod parse → `processLink` (pure-ish validation) → `createLink` (DB write + `waitUntil` fan-out) → webhook `link.created`.

### One small safe change (do not push)

In [apps/web/app/api/links/route.ts](../../apps/web/app/api/links/route.ts), find the rate-limit message at L64-L66. Reword it. Then run (both __inferred__):

```bash
cd apps/web
npx tsc --noEmit          # typecheck; the repo has no dedicated typecheck script — lint is `next lint`
pnpm test tests/links/create-link.test.ts   # will fail without E2E_BASE_URL/E2E_TOKEN — understand *why* from tests/utils/env.ts
```

The lesson is the failure itself: this repo's "tests" are integration tests against a deployed instance. That's a real architectural choice with tradeoffs — see [05-quality-engineering/01-testing-strategy.md](05-quality-engineering/01-testing-strategy.md).

### Teach-back (the interview rep)

Set a 3-minute timer. Aloud, no notes: *"Walk me through what happens when someone clicks a Dub short link."* You pass if you mention: hostname dispatch, cache tiers with TTLs, the not-found/expired/password guards, that click recording is fire-and-forget (`waitUntil`), and that analytics events go to a columnar store while MySQL keeps denormalized counters. Then do it again in 90 seconds. That compression skill is the interview.

## What the fast track skips

Partner programs and payouts (`app/(ee)`), the folder permission system, OAuth apps and integrations, the email package, the embeds SDK, Stripe billing, and the entire dashboard UI layer. The full curriculum covers the load-bearing parts of these; start with [01-codebase-cartography/01-system-map.md](01-codebase-cartography/01-system-map.md).
