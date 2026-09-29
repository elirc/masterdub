# System Design From This Repo: "Design a Link Shortener with Analytics"

This is the single most common system-design prompt for mid-level fullstack candidates — and you are sitting inside a production-grade answer. Work through this as a whiteboard exercise: at each step, sketch your own answer first, then compare with what Dub actually chose (anchors verified 2026-07-09). Cross-reference [03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md) for the honest tradeoffs.

## Step 0 — Requirements (2–3 minutes, out loud)

Functional: create short links (custom domains, custom keys), redirect fast, record clicks, show analytics (timeseries, by country/device/referrer), track conversions (lead/sale), team workspaces, API + dashboard, webhooks.

Non-functional: redirect p99 in low tens of ms; reads outnumber writes by orders of magnitude; click volume ≫ link volume; analytics can lag by seconds; multi-tenant isolation is a hard requirement; abuse (spam/phishing links) is an operational reality.

- **Junior answer sounds like:** jumping straight to "hash the URL, store in a table."
- **Mid-level adds:** read/write asymmetry, explicit latency budget for the redirect, tenancy.
- **Senior includes:** abuse/moderation as a first-class requirement (Dub has a banned-links path — [lib/middleware/link.ts#L212-L221](../../../apps/web/lib/middleware/link.ts#L212-L221) — and blacklist checks in [process-link.ts#L1](../../../apps/web/lib/api/links/process-link.ts#L1)), plus privacy constraints (EU IP redaction, [record-click.ts#L143-L145](../../../apps/web/lib/tinybird/record-click.ts#L143-L145)).

## Step 1 — API sketch

Your sketch should include: `POST /links`, `GET /links`, `PATCH/DELETE /links/:id`, `GET /analytics`, `POST /track/lead`, `POST /track/sale`, webhook registration.

What Dub chose: exactly that, as REST under [app/api/links/route.ts](../../../apps/web/app/api/links/route.ts), [app/api/analytics/route.ts](../../../apps/web/app/api/analytics/route.ts), [app/(ee)/api/track/](../../../apps/web/app/(ee)/api/track/) — with Zod schemas generating OpenAPI ([lib/zod/schemas/links.ts](../../../apps/web/lib/zod/schemas/links.ts), [scripts/generate-openapi.ts](../../../apps/web/scripts/generate-openapi.ts)). Auth is workspace API keys (hashed, scoped, plan-rate-limited) via [withWorkspace](../../../apps/web/lib/auth/workspace.ts#L58-L80).

Stronger/simpler alternative to mention: tRPC or GraphQL for the dashboard — Dub instead shares REST between dashboard and public API. Tradeoff: one contract to maintain vs REST's verbosity for UI needs (they compensate with `includeUser=true`-style query flags, [use-links.ts#L33](../../../apps/web/lib/swr/use-links.ts#L33)).

- **Junior:** lists endpoints.
- **Mid-level:** specifies auth model (workspace-scoped keys, not user keys — Dub actively migrates users off personal keys, see the error message at [workspace.ts#L155-L159](../../../apps/web/lib/auth/workspace.ts#L155-L159)), idempotent upsert (`PUT /links/upsert` exists — [app/api/links/upsert/](../../../apps/web/app/api/links/upsert/)), pagination.
- **Senior:** rate limits per plan surfaced in headers ([workspace.ts#L248-L258](../../../apps/web/lib/auth/workspace.ts#L248-L258)), error taxonomy with doc URLs ([lib/api/errors.ts#L43-L60](../../../apps/web/lib/api/errors.ts#L43-L60)), API versioning strategy (Dub uses in-place deprecation shims — [analytics route L27-L31](../../../apps/web/app/api/analytics/route.ts#L27-L31)).

## Step 2 — Data model

Sketch: `Workspace (Project)`, `User`, `Link`, `Domain`, `Tag`, click events. The interviewer wants: where does the `(domain, key) → url` mapping live, and where do events live?

What Dub chose ([packages/prisma/schema/link.prisma](../../../packages/prisma/schema/link.prisma)):

- `Link` with `@@unique([domain, key])` (L95) — the lookup key; plus `@@unique([projectId, externalId])` (L96) for customer-side idempotency; `shortLink` denormalized as its own unique column (L6).
- **Denormalized counters on the row** — `clicks`, `leads`, `sales`, `saleAmount` (L56-L61) — so lists and quotas never touch the event store.
- **Events in Tinybird** (columnar), not MySQL: [lib/tinybird/record-click.ts#L179-L189](../../../apps/web/lib/tinybird/record-click.ts#L179-L189). MySQL would die at click volume; columnar stores make `GROUP BY country` cheap.
- Key generation: random `nanoid` with collision check (`getRandomKey` in [lib/planetscale](../../../apps/web/lib/planetscale/)), *not* an auto-increment base62 encode. Mention both; random keys avoid enumeration of other tenants' links (security), at the cost of collision retries.

- **Junior:** one table with a `clicks` int.
- **Mid-level:** separates event store from row counters and can say why; names the composite unique indexes and what query each serves (the schema comments do this explicitly — L95-L105).
- **Senior:** discusses index-per-query-pattern discipline, `BigInt` for money-in-cents (L61), soft state like `disabledAt`/`expiresAt` on the row so the redirect path needs no joins, and the drift risk of counters vs events.

## Step 3 — The redirect hot path

This is where the interview is won. Sketch: request → cache → 302.

What Dub chose ([full trace in key-flows Flow 1](../01-codebase-cartography/05-key-flows.md)):

1. Hostname-routed middleware, `runtime: "nodejs"` ([middleware.ts#L21-L33](../../../apps/web/middleware.ts#L21-L33)).
2. Three cache tiers: per-instance LRU (10k entries, **5s TTL** — a stampede breaker, not a cache), Redis (24h), Vercel runtime cache as Redis-outage fallback ([cache.ts#L14-L30](../../../apps/web/lib/api/links/cache.ts#L14-L30)).
3. DB fallback via PlanetScale **HTTP driver** (no connection pool pain in serverless) — [link.ts#L88-L92](../../../apps/web/lib/middleware/link.ts#L88-L92).
4. **302 not 301** for normal links ([link.ts#L575](../../../apps/web/lib/middleware/link.ts#L575)) — 301s get cached by browsers/CDNs and you lose click tracking and the ability to change the destination. Root-domain links get 301. Saying this unprompted is a strong signal.
5. All click recording via `waitUntil` — zero analytics latency in the response.

Variation the interviewer will probe: **cache invalidation on link edit**. Dub: write-through set + LRU update + `revalidateTag` ([cache.ts#L50-L64](../../../apps/web/lib/api/links/cache.ts#L50-L64)); staleness bounded by 5s LRU TTL on *other* instances.

## Step 4 — Click ingestion and analytics

Sketch: redirect → async event pipeline → columnar store; counters updated separately.

What Dub chose ([record-click.ts](../../../apps/web/lib/tinybird/record-click.ts)): bot filtering, 1-hour dedup per (link, visitor-identity) in Redis, GDPR IP redaction, then a `Promise.allSettled` fan-out — Tinybird HTTP ingest with retry, atomic `UPDATE Link SET clicks = clicks + 1`, workspace usage via Redis stream with direct-SQL fallback, webhooks via QStash.

Stronger alternative to discuss: Kafka/Kinesis + consumer for exactly-once-ish processing and replay. Dub's choice (HTTP ingest + fire-and-forget) is simpler and loses events on edge failures — acceptable because click analytics tolerates small loss; billing counters get the fallback path instead. Being able to say *which data can be lossy* is the senior move.

Conversion tracking: the click sets a `dub_id` cookie ([link.ts#L254-L279](../../../apps/web/lib/middleware/link.ts#L254-L279)); later `/track/lead` joins the conversion to the click. Because Tinybird ingestion isn't immediately queryable, click data is cached in Redis for 5 minutes ([record-click.ts#L171-L175](../../../apps/web/lib/tinybird/record-click.ts#L171-L175)) — a lovely concrete detail about **read-your-writes** limitations of analytics stores.

## Step 5 — Multi-tenancy and authorization

What Dub chose: single database, `projectId` column tenancy; every resource lookup takes `workspaceId` ([get-link-or-throw](../../../apps/web/lib/api/links/get-link-or-throw.ts)); role→permission RBAC with token-scope intersection ([workspace.ts#L410-L418](../../../apps/web/lib/auth/workspace.ts#L410-L418)); 404-not-403 for foreign tenants (anti-enumeration, [#L386-L389](../../../apps/web/lib/auth/workspace.ts#L386-L389)); folders add intra-workspace ACLs ([analytics route L94-L101](../../../apps/web/app/api/analytics/route.ts#L94-L101)).

Alternatives: schema-per-tenant (operational pain at Dub's tenant count), row-level security in Postgres (they're on MySQL/PlanetScale). Column tenancy + scoped-helpers convention is the industry default; its weakness is that it's convention — one raw query can bypass it.

## Step 6 — Scaling concerns, stated as numbers

- Redirects scale horizontally: stateless middleware + Redis; LRU absorbs per-instance hot keys.
- Writes (link CRUD) are low volume — MySQL fine.
- Clicks: Tinybird ingest is append-only; MySQL sees only 1 UPDATE per click (and dedup caps that at 1/hour/visitor/link).
- The webhook usage-limit check does read MySQL per clicked webhook-link ([record-click.ts#L266-L279](../../../apps/web/lib/tinybird/record-click.ts#L266-L279)) — a hot-path read you'd cache next.

## Variation prompts (practice each for 5 minutes aloud)

1. **"Now add real-time analytics on the dashboard."** Discuss: polling SWR (current), Tinybird query latency, websockets/SSE tradeoff, and why 30s staleness is probably fine — push back on the requirement.
2. **"Now support 10x click traffic."** LRU TTL tuning, Redis cluster, regional Tinybird ingest endpoints, drop the per-click webhook usage-limit DB read, sample the dedup cache.
3. **"Now add multi-region."** Redirect data (Redis link cache) must replicate; PlanetScale read replicas; the clickId cookie keeps working; QStash/Tinybird are already regional-agnostic HTTP.
4. **"A customer's webhook endpoint is down for 6 hours."** QStash retries → failure callback → auto-disable policy ([lib/webhook/failure.ts](../../../apps/web/lib/webhook/failure.ts)) → re-enable + replay story (Dub has no replay — propose one, it's a real gap).
5. **"Make link creation strongly consistent with the cache."** Discuss write-through (current), what happens when the Redis set fails after the DB commit (stale-miss → DB fallback saves you — the design is *self-healing on miss*, which is why TTL-bounded inconsistency is acceptable here).

## Grading yourself

Basic pass: schema + cache + 302 + async analytics. Solid (mid-level hire): all of that plus tenancy scoping, 301-vs-302 reasoning, counter-vs-event split, cache invalidation story. Strong (senior signal): lossy-vs-lossless data classification, anti-enumeration 404s, the clickId read-your-writes workaround, and at least one place you'd disagree with Dub's choices — with a cost argument (candidates who can respectfully critique a real system stand out).
