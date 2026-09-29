# Architecture Critique

An opinionated review, separated into confirmed observations (anchored) and hypotheses (labeled). This file doubles as your system-design interview prep — every judgment here is phrased the way you'd defend it on a whiteboard ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)).

## Strongest design choices

1. **Latency-tiered storage matched to data temperature.** Hot (redirect config) in LRU/Redis; warm (entities) in MySQL; cold-but-huge (events) in Tinybird. Each store does what it's structurally good at. Evidence: [cache.ts#L14-L30](../../../apps/web/lib/api/links/cache.ts#L14-L30), [record-click.ts#L179-L199](../../../apps/web/lib/tinybird/record-click.ts#L179-L199).
2. **Policy-at-the-boundary via `withWorkspace`** with declarative per-route requirements — authorization is reviewable in the route file and unforgettable by construction ([workspace.ts#L58-L80](../../../apps/web/lib/auth/workspace.ts#L58-L80)).
3. **Contracts as one artifact**: Zod → types → OpenAPI → outbound webhook validation. Drift between docs and behavior is structurally hard ([lib/zod/schemas](../../../apps/web/lib/zod/schemas/), [links route L92](../../../apps/web/app/api/links/route.ts#L92)).
4. **Self-healing cache design**: cache misses fall back to DB and repopulate; TTLs bound staleness; invalidation failures degrade to slower-but-correct. Wrong cache data has a 24h worst case, not forever ([cache.ts](../../../apps/web/lib/api/links/cache.ts)).
5. **User-latency never pays for bookkeeping**: `waitUntil` everywhere on analytics/logs/webhooks; queues for anything needing retries. The p99 discipline is consistent.
6. **Index discipline with named queries** in the schema ([link.prisma#L95-L105](../../../packages/prisma/schema/link.prisma#L95-L105)) — rare and excellent.

## Risks and tradeoffs (prioritized)

1. **No durable link between DB writes and their side effects** (no outbox). Crash/failure after `prisma.link.create` but during the fan-out loses cache priming, Tinybird metadata, webhooks — silently. Mitigations exist (TTL re-derivation, self-healing miss) for *some* effects; webhooks and Tinybird metadata have none. Confidence: confirmed by reading [create-link.ts#L142-L228](../../../apps/web/lib/api/links/create-link.ts#L142-L228); severity: medium (frequency low, detectability low).
2. **Webhook delivery silently skips on cache miss** — the TODO at [record-click.ts#L305-L307](../../../apps/web/lib/tinybird/record-click.ts#L305-L307). Confirmed code path; impact: missed customer events with zero signal.
3. **Counter/event drift with no reconciliation** (hypothesis: no reconciliation job found, but 38 cron dirs weren't exhaustively read — verify before asserting). Billing counters ride best-effort pipelines.
4. **Two parallel auth stacks** (REST wrapper vs server-action middleware) — duplicated policy, divergence risk. Confirmed structurally; whether they *have* diverged is unverified.
5. **Idempotency pushed onto webhook consumers** (no deduplicationId — [qstash.ts#L69-L71](../../../apps/web/lib/webhook/qstash.ts#L69-L71)).
6. **Implicit cache-shape contract** enforced by `as any` ([link.ts#L110](../../../apps/web/lib/middleware/link.ts#L110)) — a new redirect feature that forgets `formatRedisLink` ships a bug that only manifests on cache *hits*, the majority path, while working in every "create then immediately test" QA flow. Insidious.
7. **Env-conditional security** (QStash signature skip off-Vercel — [verify-qstash.ts#L18-L21](../../../apps/web/lib/cron/verify-qstash.ts#L18-L21)): fine today; becomes a hole the day anyone self-hosts outside Vercel.
8. **The 528-line auth god-function** — every new concern lands there; testing it in isolation is hard (and no unit tests for it were found).

## "What I'd change owning this for 3 months"

Ordered by risk-reduction per unit effort, each with migration + test strategy:

1. **Metrics on silent-skip paths** (webhook cache miss, usage-gate skip, allSettled rejections → counters, not just logs). Migration: additive, zero behavior change. Test: assert counter increments in unit tests of the skip branches. This is observability before surgery.
2. **DB fallback for the webhook cache** (resolves the TODO). Small diff, follows the existing cache-aside pattern. Test: integration test with cache flushed.
3. **deduplicationId on QStash publishes** (event id already exists in the payload). Consumer-visible improvement; document in webhook docs. Test: publish twice, assert single delivery in QStash sandbox.
4. **Reconciliation cron**: nightly sample of N links comparing `Link.clicks` vs Tinybird count, emitting drift metrics. Not a fix — a measurement that tells you whether problems 1/3 are theoretical or live. Test: seed known drift, assert detection.
5. **Outbox for webhooks only** (not for cache/Tinybird, which self-heal): write event rows in the create transaction; a publisher cron drains them to QStash. Migration: new table + dual-write behind a flag, then cut over. Test: kill the publisher mid-drain, assert no loss/dupes beyond at-least-once.
6. **Extract the token-auth block from `withWorkspace`** into a unit-testable function; add the revocation/cache-invalidation test that's currently unanswerable.

What I would *not* do: microservices, GraphQL rewrite, replacing SWR, "fixing" the snake_case leak — all churn with no risk story. Saying what you wouldn't change is half of senior judgment.

## Hypotheses needing verification before acting

- Token revocation vs Redis token-cache TTL race (read `token-cache.ts` delete paths).
- Whether `identityHash` derivation satisfies the same EU privacy bar as the IP redaction (read `get-identity-hash.ts`).
- Prod schema-change workflow (db push vs PlanetScale deploy requests).
- Whether Playwright covers the redirect decision tree.

Drill: pick risk #1 and argue the *opposite* position (outbox not worth it here) for two minutes aloud — cost of the table, ordering complexity, the fact that most effects self-heal. If you can steelman both sides with anchors, you're interview-ready on this system.
