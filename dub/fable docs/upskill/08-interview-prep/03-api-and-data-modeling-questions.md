# API and Data Modeling Question Cards

Twelve cards, all repo-anchored.

---

## Q1: How do you structure a REST API so authorization can't be forgotten?

Round: API/data
Repo anchor: [withWorkspace](../../../apps/web/lib/auth/workspace.ts#L58-L80) + declarative `requiredPermissions` ([links route L43](../../../apps/web/app/api/links/route.ts#L43), [L107-L109](../../../apps/web/app/api/links/route.ts#L107-L109)).
Junior: "check the user in each handler."
Mid adds: a wrapper every route must pass through, with per-route policy as *options* — the handler literally cannot run unauthenticated; plus scoped resource lookups as the second line.
Senior includes: the residual risks — carve-outs inside the wrapper, parallel surfaces (server actions) with separate auth, and convention-based query scoping — and how you'd audit each.
Follow-ups: "what if one route needs no auth?" (explicit allowlist, loud comments — see the anonymous carve-out).
Drill: recite the eight things `withWorkspace` checks, in order.

## Q2: Design pagination for a resource with millions of rows.

Round: API/data
Repo anchor: pagination schemas in [lib/zod/schemas/misc.ts](../../../apps/web/lib/zod/schemas/misc.ts) (`getPaginationQuerySchema` and a **cursor** variant — both exist, [links.ts imports both](../../../apps/web/lib/zod/schemas/links.ts)); `LINKS_MAX_PAGE_SIZE = 100`.
Junior: "page and limit params."
Mid adds: offset pagination degrades (OFFSET n scans n rows) and skips/dupes under concurrent writes; cursor (keyset) pagination — `WHERE createdAt < cursor ORDER BY createdAt DESC LIMIT n` — is stable and index-friendly; cap page size server-side always.
Senior includes: cursor must be a *unique* ordering (createdAt ties → compound cursor with id); the repo supports both because public APIs can't drop offset once shipped (contract gravity); count queries as the hidden cost (`/links/count` is its own endpoint here for a reason).
Drill: write the keyset WHERE clause for the links list including the tie-breaker.

## Q3: Model links, tags, and workspaces. Go.

Round: API/data (live modeling)
Repo anchor: [link.prisma](../../../packages/prisma/schema/link.prisma), [tag.prisma](../../../packages/prisma/schema/tag.prisma).
Junior: three tables, foreign keys, done.
Mid adds: the join table (`LinkTag`) for n—n; tenancy column on *both* Link and Tag with `@@unique([name, projectId])` on tags (name-unique per tenant, not global); the `(domain, key)` natural key alongside the surrogate id.
Senior includes: which uniques double as idempotency keys ([externalId per workspace](../../../packages/prisma/schema/link.prisma#L96)); index-per-query discipline; where denormalization is allowed to creep in (counters, `shortLink`) and what maintains it.
Follow-ups: "user-supplied external ids — why scope their uniqueness?" (kata 5's cross-tenant lesson).
Drill: whiteboard it in 5 minutes, then diff against the real schema; note every field you forgot and *why it exists*.

## Q4: What's IDOR and how do you make it structurally impossible?

Round: API/data / security
Repo anchor: [get-link-or-throw.ts](../../../apps/web/lib/api/links/get-link-or-throw.ts); 404-not-403 at [workspace.ts#L386-L389](../../../apps/web/lib/auth/workspace.ts#L386-L389).
Junior: "check the user owns the resource."
Mid adds: the *helper-signature* trick — lookups take `workspaceId` as a required parameter, so the safe query is the easy query; plus 404 for foreign resources to prevent enumeration.
Senior includes: "structurally impossible" is aspirational — raw queries bypass conventions; defenses in depth: scoped helpers + review checklist + tests asserting cross-tenant 404 ([folder-link-access.test.ts](../../../apps/web/tests/links/folder-link-access.test.ts) as the pattern) + random non-sequential ids (cuid/nanoid here) as the last-ditch layer.
Drill: say the 404-vs-403 argument in two sentences.

## Q5: How do API keys work in a well-built SaaS?

Round: API/data / security
Repo anchor: [workspace.ts#L175-L309](../../../apps/web/lib/auth/workspace.ts#L175-L309); [hash-token.ts](../../../apps/web/lib/auth/hash-token.ts); scopes in [lib/api/tokens/scopes.ts](../../../apps/web/lib/api/tokens/scopes.ts).
Junior: "a secret in the Authorization header checked against the DB."
Mid adds: stored hashed; prefix conventions (`dub_`) to distinguish token classes; workspace-scoped tokens with *scopes intersected against the user's role*; per-plan rate limits keyed by token; lastUsed tracking (throttled).
Senior includes: the cache-revocation race (cached token vs deleted token — TTL bounds it; invalidate-on-delete removes it — investigate which this repo does); machine users for integration-owned resources and their privilege implications ([#L404-L408](../../../apps/web/lib/auth/workspace.ts#L404-L408)).
Drill: design the token table columns from memory, then compare with [token.prisma](../../../packages/prisma/schema/token.prisma).

## Q6: How do you evolve a public API without breaking clients?

Round: API/data
Repo anchor: deprecation shims at [analytics route L27-L31, L111-L117](../../../apps/web/app/api/analytics/route.ts#L27-L31); field aliases at [track/lead L17-L21](../../../apps/web/app/(ee)/api/track/lead/route.ts#L17-L21); [deprecated.ts](../../../apps/web/lib/zod/schemas/deprecated.ts).
Junior: "version the URL, /v2."
Mid adds: additive-first (new optional fields never break); in-place deprecation with aliasing at the boundary so the core sees one shape; OpenAPI/SDK regeneration as part of the change; telemetry on deprecated usage before removal.
Senior includes: URL versioning's real cost (N live code paths forever, SDK matrix); contract tests as the enforcement (`toStrictEqual` against expected shapes in this repo's suite pins responses *exactly*); a sunset policy is a people process, not a code process.
Drill: kata 8 (the versioning RFC).

## Q7: When do you denormalize, and what's the price?

Round: API/data
Repo anchor: `clicks/leads/sales/saleAmount` counters + `lastClicked` on Link ([link.prisma#L56-L65](../../../packages/prisma/schema/link.prisma#L56-L65)); maintained at [record-click.ts#L194-L199](../../../apps/web/lib/tinybird/record-click.ts#L194-L199); `shortLink` as second representation of `(domain,key)`.
Junior: "denormalize for speed."
Mid adds: denormalize when the read pattern (list 100 links with click counts) can't afford the aggregate (COUNT over a billion-row event store); the price is a *maintenance obligation* — every writer must update it — and drift when writers are best-effort.
Senior includes: classify each denormalized field by its repair story: `shortLink` is consistent-by-construction (same transaction), counters are eventually-consistent-with-drift (no reconciliation — the repo's honest gap); billing-relevant counters deserve better than vanity counters, even on the same mechanism.
Drill: for each denormalized field on Link, name its writer(s) and repair story.

## Q8: How do you change a production schema safely?

Round: API/data
Repo anchor: [02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md) — expand-migrate-contract; the repo's raw-SQL sites as the hidden consumers ([record-click.ts#L196-L199](../../../apps/web/lib/tinybird/record-click.ts#L196-L199)).
Junior: "write a migration and run it."
Mid adds: additive → backfill → enforce → contract, each step deploy-separated; never rename in place; code must work with both shapes during the window; rollback means *code* rollback stays safe against the *new* schema.
Senior includes: ORM-invisible consumers (raw SQL, cached shapes in Redis, event-store copies in Tinybird — this repo has all three for Link); lock behavior of DDL at scale (PlanetScale's online DDL as the platform answer); and migrations as the reason `BigInt`-vs-`Int` decisions matter on day one.
Drill: the `clickLimit` design from [03/02's drill](../03-architecture-and-patterns/02-data-model-and-persistence.md), aloud, five minutes.

## Q9: Design webhook delivery. What makes it hard?

Round: API/data / system design bridge
Repo anchor: [qstash.ts](../../../apps/web/lib/webhook/qstash.ts); [failure.ts](../../../apps/web/lib/webhook/failure.ts); flow 6 in [key flows](../01-codebase-cartography/05-key-flows.md).
Junior: "POST the JSON to their URL."
Mid adds: their server is down/slow/hostile — so: queue with retries+backoff, HMAC signatures, delivery timeouts, per-endpoint failure tracking with auto-disable, event log for support.
Senior includes: idempotency (dedup ids — the repo's open TODO), ordering (mostly *don't promise it* — say so in docs), replay, and the producer-side contract (Zod-parse before send so you never emit an invalid payload — [links route L92](../../../apps/web/app/api/links/route.ts#L92)).
Drill: the consumer-side answer too — verify signature, 2xx fast, process async, dedupe by event id.

## Q10: Rate limiting — design and edge cases.

Round: API/data
Repo anchor: plan-based limits ([get-ratelimit-for-plan.ts](../../../apps/web/lib/api/get-ratelimit-for-plan.ts), applied at [workspace.ts#L237-L265](../../../apps/web/lib/auth/workspace.ts#L237-L265)); per-second analytics limits; anonymous 10/day/IP ([links route L58-L69](../../../apps/web/app/api/links/route.ts#L58-L69)); the rate-limiter-as-write-throttle trick ([#L272-L300](../../../apps/web/lib/auth/workspace.ts#L272-L300)).
Junior: "X requests per minute per user."
Mid adds: the identity question (per token? user? IP? — this repo uses all three for different surfaces); sliding vs fixed windows; return 429 *with headers* so clients can behave; different limits per endpoint class (analytics tighter).
Senior includes: shared-IP collateral damage (NAT/CDN — the anonymous-creation scenario in [debugging scenario 5](../05-quality-engineering/03-systematic-debugging.md)); limits as a monetization surface (per-plan); and creative reuse (limiting *your own writes*).
Drill: name the three identity keys this repo rate-limits on and one failure mode of each.

## Q11: Transactions — when do you actually need one?

Round: API/data
Repo anchor: nested-write create as an implicit transaction ([create-link.ts#L56-L138](../../../apps/web/lib/api/links/create-link.ts#L56-L138)); the deliberate *absence* of cross-store transactions (fan-out after).
Junior: "wrap related writes in a transaction."
Mid adds: transactions give atomicity within one database — the link+tags+dashboard create is all-or-nothing; they cannot span MySQL+Redis+Tinybird, so cross-store consistency needs a different tool (TTL self-healing, outbox, sagas); long transactions hold locks — keep them tight.
Senior includes: the classification habit — for each multi-write operation ask "same DB? then transaction. Different stores? then pick: eventual+repair, outbox, or redesign so one store owns truth"; and interactive-transaction pitfalls in serverless (connection lifetimes).
Drill: list three multi-write operations in this repo and classify each per the habit.

## Q12: How would you add caching to an endpoint, end to end?

Round: API/data
Repo anchor: the full worked example — [cache.ts](../../../apps/web/lib/api/links/cache.ts) tiers, write-through at [create-link.ts#L149-L155](../../../apps/web/lib/api/links/create-link.ts#L149-L155), invalidation questions in [debugging scenario 1](../05-quality-engineering/03-systematic-debugging.md).
Junior: "put Redis in front of the query."
Mid adds: the four decisions — key (must include every input: the case-sensitivity lesson), TTL (staleness budget), invalidation (write-through vs delete-on-write vs TTL-only), and stampede behavior (this repo's 5s LRU); plus "what does a hit skip?" (authorization must NOT be skippable — cache *data*, not *decisions*, unless scoped per principal).
Senior includes: negative caching (the not-found rewrite is CDN-cached via cache tags here — [link.ts#L65-L71](../../../apps/web/lib/middleware/link.ts#L65-L71)); cache-shape drift as the insidious failure ([critique #6](../03-architecture-and-patterns/06-architecture-critique.md)); and the self-healing property that makes TTL-bounded staleness acceptable.
Drill: apply the four decisions to caching `GET /api/workspaces/[id]` — write each answer in one line.
