# Pattern Catalog

Sixteen patterns this repo actually uses, written for recognition — so you can spot the same shape in any codebase and name it in an interview. Anchors verified 2026-07-09.

---

## Pattern 1: Higher-order route wrapper (policy at the boundary)

Problem it solves: every API route needs the same authn → rate limit → tenant → permission → plan pipeline; copy-pasting it guarantees someone forgets a step.

General shape: `export const GET = withPolicy(handler, options)` — a function that takes a handler and returns a hardened handler.

Real example: [apps/web/lib/auth/workspace.ts#L58-L80](../../../apps/web/lib/auth/workspace.ts#L58-L80) (`withWorkspace`), consumed at [app/api/links/route.ts#L20-L45](../../../apps/web/app/api/links/route.ts#L20-L45).
Second example: the sibling wrappers in [lib/auth/index.ts](../../../apps/web/lib/auth/index.ts) (`withSession`, `withAdmin` — see [lib/auth/admin.ts](../../../apps/web/lib/auth/admin.ts)).

Why this implementation works: options (`requiredPermissions`, `requiredPlan`, `requiredRoles`, `featureFlag`) make each route's policy *declarative and reviewable in the route file itself*.

Failure modes: the wrapper becomes a god-function (this one is 528 lines); carve-outs inside it (anonymous link creation, [workspace.ts#L134-L146](../../../apps/web/lib/auth/workspace.ts#L134-L146)) are invisible at call sites.

Use it when: cross-cutting policy must be impossible to forget. Avoid it when: only one route needs the behavior — inline it.

Interview angle: "How do you enforce authorization consistently across an API?"

Drill: find one more wrapper in `lib/auth/` and diff its pipeline against `withWorkspace`.

---

## Pattern 2: Errors as values at the domain layer, exceptions at the edge

Problem it solves: domain validation that throws is painful in bulk operations (one bad item aborts 99 good ones).

General shape: domain function returns a discriminated union `{link, error: null} | {link, error, code}`; the HTTP layer converts to a thrown, typed error.

Real example: [lib/api/links/process-link.ts#L44-L57](../../../apps/web/lib/api/links/process-link.ts#L44-L57) (return type), converted at [app/api/links/route.ts#L77-L82](../../../apps/web/app/api/links/route.ts#L77-L82).
Second example: bulk creation consumes the same union per-item — [lib/api/links/bulk-create-links.ts](../../../apps/web/lib/api/links/bulk-create-links.ts) (existence verified by import in [lib/api/links/index.ts](../../../apps/web/lib/api/links/index.ts); read before citing lines).

Why this implementation works: `processLink` is reusable across single, bulk, upsert, and update flows with per-item error reporting.

Failure modes: callers can forget to check `error` (TypeScript narrows only if they do); mixing throw-style and value-style in one layer confuses everyone.

Use it when: the same validation feeds single and batch paths. Avoid it when: an error should always abort — just throw.

Interview angle: "Exceptions vs result types" — you have a codebase that deliberately uses both, at different layers.

Drill: trace `error` from `processLink` through the bulk endpoint's response shape.

---

## Pattern 3: Typed error taxonomy with one exit door

Problem it solves: consistent HTTP error bodies + status codes across hundreds of routes.

General shape: an error class carrying a machine code; a single `handleAndReturnErrorResponse` that maps *any* thrown thing (Zod, Prisma, custom, unknown) to the public error contract.

Real example: [lib/api/errors.ts#L43-L60](../../../apps/web/lib/api/errors.ts#L43-L60) (`DubApiError`), mapper at [#L92-L130](../../../apps/web/lib/api/errors.ts#L92-L130) (Zod → 422, `P2025` → 404).
Second example: every wrapper funnels through it — [lib/auth/workspace.ts#L502-L505](../../../apps/web/lib/auth/workspace.ts#L502-L505).

Why this implementation works: each error carries a `doc_url`; the OpenAPI spec is generated from the same Zod `ErrorSchema` ([errors.ts#L23-L38](../../../apps/web/lib/api/errors.ts#L23-L38)) — error contract and docs cannot drift.

Failure modes: catching-and-wrapping too early loses stack context; new error codes must be added to the code↔status map or they fall through to 500.

Use it when: you have a public API. Avoid it when: internal-only service where a plain 500 + log suffices.

Interview angle: "How do you design API error responses?"

Drill: follow a Prisma P2025 from `prisma.link.update` to the JSON body a client sees.

---

## Pattern 4: Tiered read cache (LRU → Redis → durable)

Problem it solves: redirect latency and Redis load during traffic spikes on hot links.

General shape: in-process LRU with tiny TTL → shared Redis with long TTL → database as truth; each miss falls through and repopulates the tier above.

Real example: [lib/api/links/cache.ts#L14-L30](../../../apps/web/lib/api/links/cache.ts#L14-L30) (10k-entry LRU, 5s TTL; Redis 24h; Vercel runtime cache 5m as Redis-outage fallback), read path [#L66-L100](../../../apps/web/lib/api/links/cache.ts#L66-L100).
Second example: token cache for API keys — [lib/auth/token-cache.ts](../../../apps/web/lib/auth/token-cache.ts) (single Redis tier; contrast is instructive).

Why this implementation works: the 5s LRU absorbs stampedes on viral links while bounding staleness to 5s; write path updates LRU synchronously *before* Redis ([#L50-L64](../../../apps/web/lib/api/links/cache.ts#L50-L64)) to prevent stale reads in the same instance.

Failure modes: in-process caches diverge across instances (each serverless instance has its own LRU); cache shape (`formatRedisLink`) silently drops fields the redirect later needs.

Use it when: read-heavy, tolerant of seconds of staleness. Avoid it when: reads must reflect writes immediately across instances.

Interview angle: "Add caching to a hot endpoint — walk me through the tiers and TTLs."

Drill: compute worst-case staleness a user can observe after editing a link, per tier.

---

## Pattern 5: Fire-and-forget with `waitUntil`

Problem it solves: analytics, cache writes, logs, and webhooks must not add latency to the user's response.

General shape: `waitUntil(promise)` — serverless runtime keeps the instance alive after the response is sent.

Real example: click recording after redirect — [lib/middleware/link.ts#L553-L567](../../../apps/web/lib/middleware/link.ts#L553-L567).
Second example: request logging in the auth wrapper — [lib/auth/workspace.ts#L486-L499](../../../apps/web/lib/auth/workspace.ts#L486-L499); post-create fan-out — [lib/api/links/create-link.ts#L142-L228](../../../apps/web/lib/api/links/create-link.ts#L142-L228).

Why this implementation works: the user's redirect is never blocked by Tinybird/Redis/webhook latency.

Failure modes: **failures are invisible to the caller** — no retry, no alert unless you add one; work is lost if the runtime kills the instance; you cannot use it for anything the response depends on.

Use it when: side effect is best-effort and observable elsewhere. Avoid it when: the effect is part of the contract (billing, quota that gates the very next request).

Interview angle: "How do you keep p99 low while still recording analytics?" — and the sharp follow-up, "what happens when that background write fails?"

Drill: grep `waitUntil(` in `apps/web/lib` and classify each use as best-effort vs actually-load-bearing.

---

## Pattern 6: `Promise.allSettled` fan-out with named-operation logging

Problem it solves: one click triggers five independent writes; one failing must not stop the others, and failures must be attributable.

General shape: array of independent promises → `allSettled` → map rejects back to a parallel array of operation names → structured error log.

Real example: [lib/tinybird/record-click.ts#L177-L262](../../../apps/web/lib/tinybird/record-click.ts#L177-L262).
Second example: [lib/api/links/create-link.ts#L149-L226](../../../apps/web/lib/api/links/create-link.ts#L149-L226) (settled, without the naming).

Why this implementation works: partial success is the correct semantic — a Tinybird hiccup shouldn't lose the MySQL counter bump.

Failure modes: index-based operation names ([record-click.ts#L237-L244](../../../apps/web/lib/tinybird/record-click.ts#L237-L244)) silently mislabel when someone reorders the array — a real trap; "settled" quietly normalizes failure unless someone reads the logs.

Use it when: effects are independent and individually recoverable. Avoid it when: effects must be atomic — use a transaction or saga.

Interview angle: "`Promise.all` vs `allSettled` vs `race`" — answer with this exact file.

Drill: refactor (on paper) the index-based naming into an object array `{name, promise}` and note what it prevents.

---

## Pattern 7: Graceful degradation — queue first, direct write fallback

Problem it solves: usage counters are updated via Redis streams (batchable, cheap), but Redis can fail and billing counters must not drift silently.

General shape: `publishToStream(...).catch(() => writeDirectlyToDatabase(...))`.

Real example: workspace usage — [lib/tinybird/record-click.ts#L202-L214](../../../apps/web/lib/tinybird/record-click.ts#L202-L214); partner activity ditto [#L216-L229](../../../apps/web/lib/tinybird/record-click.ts#L216-L229).
Second example: link-usage event on create — [lib/api/links/create-link.ts#L210-L216](../../../apps/web/lib/api/links/create-link.ts#L210-L216) (investigate: this one has **no** `.catch` fallback — asymmetry worth questioning).

Why this implementation works: the stream consumer (see `app/(ee)/api/cron/streams/`) can batch UPDATEs; the fallback keeps correctness when the stream is down.

Failure modes: double-count if the publish *succeeded* but the promise rejected after (at-least-once ambiguity); fallback path exercises rarely → bit-rots.

Use it when: high-frequency counter updates. Avoid it when: exactly-once matters more than availability (money movement).

Interview angle: "How do you absorb write-heavy counters without hammering MySQL?"

Drill: find the stream consumer cron job and identify its batching window.

---

## Pattern 8: Rate limiter as write-throttle

Problem it solves: updating `token.lastUsed` on every API call would write to MySQL at API QPS.

General shape: use a distributed rate limiter (`1 per minute`) as a *gate* around a non-critical write, not around user traffic.

Real example: [lib/auth/workspace.ts#L272-L300](../../../apps/web/lib/auth/workspace.ts#L272-L300).
Second example: click dedup is the same idea inverted — one recorded click per identity per hour ([lib/tinybird/record-click.ts#L88-L108](../../../apps/web/lib/tinybird/record-click.ts#L88-L108)).

Why this implementation works: repurposes existing infra (Upstash ratelimit) instead of a new mechanism; precision of `lastUsed` only needs minutes.

Failure modes: semantics hide in a rate-limit key string; anyone grepping for "where is lastUsed throttled" won't find a config flag.

Use it when: high-frequency low-value writes. Avoid it when: the timestamp has audit/security meaning (then buffer and flush instead).

Interview angle: "How would you avoid a hot-row UPDATE on every request?"

Drill: estimate writes/day saved for a workspace doing 100 req/s.

---

## Pattern 9: Hashed credentials + cache-aside token lookup

Problem it solves: API keys must be verifiable fast but never stored or logged in plaintext.

General shape: store `hash(key)`; on request, hash the presented key, look up by hash (cache → DB), cache the token record.

Real example: [lib/auth/workspace.ts#L175-L235](../../../apps/web/lib/auth/workspace.ts#L175-L235) with [lib/auth/hash-token.ts](../../../apps/web/lib/auth/hash-token.ts).
Second example: webhook secrets sign payloads instead of being sent — [lib/webhook/signature.ts](../../../apps/web/lib/webhook/signature.ts).

Why this implementation works: DB compromise doesn't leak usable keys; Redis cache keeps auth off MySQL for hot tokens.

Failure modes: **revocation vs cache TTL** — a deleted token may keep working until the cache entry expires unless deletion invalidates the cache (investigate [lib/auth/token-cache.ts](../../../apps/web/lib/auth/token-cache.ts) delete path before claiming either way).

Use it when: any bearer-credential system. Avoid it when: never — this is table stakes; the variable is the caching.

Interview angle: "How do you store API keys?" — an instant-fail question if you say plaintext; a strong answer discusses the revocation/cache race.

Drill: write the timeline of a token revoked at T=0 with a cached entry set at T-1min; when does it stop working?

---

## Pattern 10: Tenant-scoped resource lookup helpers

Problem it solves: IDOR — user A fetching user B's link by guessing an id.

General shape: never `findUnique({id})` from request input; always `getXOrThrow({workspaceId, id})` so tenant scope is a *required parameter of the helper's signature*.

Real example: [lib/api/links/get-link-or-throw.ts](../../../apps/web/lib/api/links/get-link-or-throw.ts) used by analytics at [app/api/analytics/route.ts#L72-L78](../../../apps/web/app/api/analytics/route.ts#L72-L78).
Second example: [lib/api/programs/get-program-or-throw.ts](../../../apps/web/lib/api/programs/get-program-or-throw.ts) at [route.ts#L62-L65](../../../apps/web/app/api/analytics/route.ts#L62-L65).

Why this implementation works: makes the unsafe query hard to write; reviewers only need to check the helper once.

Failure modes: raw `prisma.link.findUnique` sprinkled in new code bypasses it — the pattern is convention, not compiler-enforced.

Use it when: multi-tenant anything. Avoid it when: genuinely global resources.

Interview angle: "What's IDOR and how do you prevent it structurally, not by remembering?"

Drill: grep `prisma.link.find` under `app/api/` and audit each hit for workspace scoping.

---

## Pattern 11: Zod schemas as the single contract (validation + types + OpenAPI)

Problem it solves: request validation, TypeScript types, and API docs drifting apart.

General shape: one Zod schema per resource in a central directory; routes `parse` at the boundary; `z.infer` provides types; OpenAPI is generated from the same schemas.

Real example: [lib/zod/schemas/links.ts](../../../apps/web/lib/zod/schemas/links.ts) (65 schema files in [lib/zod/schemas/](../../../apps/web/lib/zod/schemas/)), consumed at [app/api/links/route.ts#L54-L56](../../../apps/web/app/api/links/route.ts#L54-L56); OpenAPI generation via the `generate-openapi` script (its `scripts/generate-openapi.ts` entry file is missing from this snapshot) and [lib/openapi/](../../../apps/web/lib/openapi/).
Second example: **outbound** contract enforcement — webhook payload parsed before send, [app/api/links/route.ts#L92](../../../apps/web/app/api/links/route.ts#L92).

Why this implementation works: the schema is the source of truth for three artifacts; changing it forces all three to move together.

Failure modes: `.passthrough()`/`any` escape hatches; schema-parse cost on hot paths; async refinements (`parseAsync`) hiding DB calls inside "validation."

Use it when: public API in TS. Avoid it when: internal hot paths where the shape is already guaranteed (parse once at the edge, trust inside).

Interview angle: "How do you keep API docs in sync with behavior?"

Drill: find where `createLinkBodySchemaAsync` differs from `createLinkBodySchema` and why the async variant exists.

---

## Pattern 12: Event store + denormalized read counters (CQRS-lite)

Problem it solves: dashboards need instant `clicks` numbers; analytics needs full event granularity; MySQL can't cheaply do both.

General shape: append every event to a columnar store (Tinybird); maintain small denormalized counters on the row (`Link.clicks`, `Project.usage`) for list views and quota checks.

Real example: [lib/tinybird/record-click.ts#L179-L199](../../../apps/web/lib/tinybird/record-click.ts#L179-L199) (event ingest + `UPDATE Link SET clicks = clicks + 1` in the same fan-out); counters declared at [packages/prisma/schema/link.prisma#L56-L61](../../../packages/prisma/schema/link.prisma#L56-L61).
Second example: partner enrollment counters — [record-click.ts#L216-L229](../../../apps/web/lib/tinybird/record-click.ts#L216-L229).

Why this implementation works: list views never touch Tinybird; quota checks are one indexed row read.

Failure modes: drift between counters and events (no reconciliation observed — possible risk); counters are approximations and must never be billed as truth without a repair job.

Use it when: high-volume events + cheap aggregate reads. Avoid it when: low volume — just `COUNT(*)`.

Interview angle: "CQRS" without the buzzword: "we keep read models next to the row and events in a columnar store."

Drill: list every place `Link.clicks` is displayed vs every place Tinybird is queried — when could a user see the two disagree?

---

## Pattern 13: Queue-mediated webhooks with signatures and result callbacks

Problem it solves: delivering webhooks synchronously couples your latency and uptime to your customers' servers.

General shape: publish signed payload to a queue (QStash); the queue retries; success/failure callbacks hit your own API to record outcomes and disable chronically failing endpoints.

Real example: [lib/webhook/qstash.ts#L39-L106](../../../apps/web/lib/webhook/qstash.ts#L39-L106); failure policy in [lib/webhook/failure.ts](../../../apps/web/lib/webhook/failure.ts).
Second example: cron jobs use the same QStash trust boundary — [lib/cron/verify-qstash.ts](../../../apps/web/lib/cron/verify-qstash.ts).

Why this implementation works: retries/backoff are the queue's job; per-webhook HMAC secret lets consumers authenticate you.

Failure modes: no `deduplicationId` yet ([qstash.ts#L69-L71](../../../apps/web/lib/webhook/qstash.ts#L69-L71)) → consumers can see duplicates; signature verification skipped off-Vercel ([verify-qstash.ts#L18-L21](../../../apps/web/lib/cron/verify-qstash.ts#L18-L21)) — fine locally, but the pattern of env-conditional security deserves scrutiny.

Use it when: any outbound webhook system. Avoid it when: internal service-to-service where a queue you own (or the DB outbox) is simpler.

Interview angle: "Design a webhook delivery system" — a complete reference implementation including the disable-on-failure policy.

Drill: read `failure.ts` and state the exact threshold at which a webhook is auto-disabled.

---

## Pattern 14: Plan and feature gating at the boundary

Problem it solves: monetization rules scattered through business logic become unenforceable.

General shape: declarative `requiredPlan` on the route wrapper; fine-grained checks (`proFeaturesCheck`) inside domain validation; beta features behind Edge Config flags.

Real example: [lib/auth/workspace.ts#L441-L473](../../../apps/web/lib/auth/workspace.ts#L441-L473) (flag, plan, free-plan analytics block); [lib/api/links/plan-features-check.ts](../../../apps/web/lib/api/links/plan-features-check.ts) used from [process-link.ts#L21](../../../apps/web/lib/api/links/process-link.ts#L21).
Second example: date-range limits per plan — [lib/api/utils/assert-valid-date-range-for-plan.ts](../../../apps/web/lib/api/utils/assert-valid-date-range-for-plan.ts) at [analytics route L103-L109](../../../apps/web/app/api/analytics/route.ts#L103-L109).

Why this implementation works: pricing pages and code stay alignable because gates are named and greppable.

Failure modes: two sources of truth (wrapper allowlist vs in-domain checks) can disagree; downgrade paths (what happens to existing pro-only links on a free plan?) are the classic untested branch.

Use it when: SaaS tiers exist. Avoid it when: flags would do (temporary rollout ≠ pricing).

Interview angle: "How do you implement plan-based feature gating without if-statements everywhere?"

Drill: pick one Pro feature and list every file enforcing it; check for drift.

---

## Pattern 15: Deprecated-contract shims

Problem it solves: public APIs accumulate old shapes you can't break.

General shape: accept old params/paths, remap to the new model at the top of the handler, mark with comments, keep telemetry on usage.

Real example: [app/api/analytics/route.ts#L27-L31](../../../apps/web/app/api/analytics/route.ts#L27-L31) and [#L111-L117](../../../apps/web/app/api/analytics/route.ts#L111-L117); deprecated field aliases in track-lead ([app/(ee)/api/track/lead/route.ts#L17-L21](../../../apps/web/app/(ee)/api/track/lead/route.ts#L17-L21)).
Second example: `lib/zod/schemas/deprecated.ts` exists for exactly this ([lib/zod/schemas/deprecated.ts](../../../apps/web/lib/zod/schemas/deprecated.ts)).

Why this implementation works: remapping at the boundary keeps the core clean — the domain never sees old shapes.

Failure modes: shims outlive their telemetry; nobody knows if removal is safe. Senior move: pair every shim with usage metrics and a sunset date.

Use it when: versioning a public API in place. Avoid it when: internal API — just migrate callers.

Interview angle: "How do you evolve an API without breaking clients?"

Drill: propose the measurement you'd add before deleting the clicks-endpoint shim.

---

## Pattern 16: ORM with escape hatches (Prisma + raw SQL on hot paths)

Problem it solves: Prisma's connection model and query shape are wrong for serverless hot paths and atomic counter bumps.

General shape: Prisma for CRUD; the PlanetScale HTTP driver (`conn.execute`) for hot-path reads/increments; retry wrapper for transient Prisma failures.

Real example: [lib/tinybird/record-click.ts#L194-L199](../../../apps/web/lib/tinybird/record-click.ts#L194-L199) (comment explains the pooling reason); edge link lookup [lib/planetscale/](../../../apps/web/lib/planetscale/); `withPrismaRetry` at [lib/api/links/create-link.ts#L55](../../../apps/web/lib/api/links/create-link.ts#L55).
Second example: webhook link+tags join as raw `JSON_ARRAYAGG` SQL — [record-click.ts#L324-L348](../../../apps/web/lib/tinybird/record-click.ts#L324-L348).

Why this implementation works: keeps ORM ergonomics for the 95% and pays the raw-SQL tax only where measured need exists.

Failure modes: raw SQL bypasses Prisma types (the `as any` at [#L344](../../../apps/web/lib/tinybird/record-click.ts#L344)); schema changes silently break raw strings; two query layers to audit for injection (parameterized `?` placeholders here — correct).

Use it when: measured hot paths. Avoid it when: "I just prefer SQL" — consistency has value.

Interview angle: "When would you drop below your ORM?" — answer with the pooling comment and the atomic increment.

Drill: verify every `conn.execute` in `record-click.ts` uses parameter placeholders, never string interpolation.
