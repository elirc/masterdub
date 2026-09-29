# Key Flows

Seven end-to-end traces. These are the repo's load-bearing paths; the rest of the curriculum rotates examples across them. Line anchors verified 2026-07-09 (see [verification log](../09-reference/verification-log.md)).

---

## Flow 1: Short-link redirect (`GET dub.sh/github`)

Why this flow matters: it is the product. It runs on every click at redirect-latency budgets, so it concentrates the repo's best thinking about caching, cache invalidation, and moving work off the hot path.

Open these files first:
- [apps/web/middleware.ts#L35-L90](../../../apps/web/middleware.ts#L35-L90) — hostname dispatch: app/API/admin/partners hostnames get their own middleware; everything else is a short link.
- [apps/web/lib/middleware/link.ts](../../../apps/web/lib/middleware/link.ts) — the whole redirect decision tree.
- [apps/web/lib/api/links/cache.ts#L14-L30](../../../apps/web/lib/api/links/cache.ts#L14-L30) — cache tiers.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Edge/middleware | [middleware.ts#L36](../../../apps/web/middleware.ts#L36) | `parse(req)` extracts `domain`, `key`, `fullKey` | strings | wrong host parsing behind proxies |
| 2 | LinkMiddleware | [link.ts#L48-L63](../../../apps/web/lib/middleware/link.ts#L48-L63) | punycode-encode key, lowercase unless case-sensitive domain, `""` → `_root`, trailing `+` = inspect mode | normalized `key` | case-sensitivity is per-domain — miss it and two links collide |
| 3 | LinkCache | [cache.ts#L66-L100](../../../apps/web/lib/api/links/cache.ts#L66-L100) | LRU (5s) → Redis (24h) → Vercel cache (5m) lookup | `RedisLinkProps` (trimmed link) | stale cache after link edit; Redis timeout path |
| 4 | DB fallback | [link.ts#L88-L132](../../../apps/web/lib/middleware/link.ts#L88-L132) | cache miss → `getLinkViaEdge` (PlanetScale HTTP driver); repopulate cache inside `ev.waitUntil` | full row | not-found → rewrite to `/[domain]/notfound` |
| 5 | Guards | [link.ts#L175-L252](../../../apps/web/lib/middleware/link.ts#L175-L252) | inspect → password → banned → disabled → expired, in that order | — | order is a contract: password check happens before click recording |
| 6 | Click identity | [link.ts#L254-L272](../../../apps/web/lib/middleware/link.ts#L254-L272) | `dub_id_<domain>_<key>` cookie else Redis recordClickCache else `nanoid(16)` | `clickId` | identity continuity across redirect → conversion |
| 7 | Targeting | [link.ts#L312-L545](../../../apps/web/lib/middleware/link.ts#L312-L545) | bot+proxy → OG proxy page; custom URI scheme; cloaking (`rewrite`); iOS/Android; geo | branch per device/country | every branch must remember `recordClick` + cookies — repetition invites drift |
| 8 | Redirect | [link.ts#L547-L579](../../../apps/web/lib/middleware/link.ts#L547-L579) | `getFinalUrl` (passthrough query, `via` for partner links), 302 (301 for `_root`), `ev.waitUntil(recordClick(...))` | 30x response | click recording is fire-and-forget: user never waits for analytics |

Validation and authorization: none in the classic sense — this is a **public, unauthenticated** surface. The "authorization" is existence + state of the link row: password ([#L188-L210](../../../apps/web/lib/middleware/link.ts#L188-L210)), banned workspace ([#L213-L221](../../../apps/web/lib/middleware/link.ts#L213-L221), `LEGAL_WORKSPACE_ID`), disabled, expired. That inversion — public read path, state-based gating — is worth saying out loud in interviews.

Persistence and side effects: reads Redis + MySQL; writes happen only via `ev.waitUntil` → [recordClick](../../../apps/web/lib/tinybird/record-click.ts) (Flow 3).

Tests that cover it: no direct middleware unit tests found under `apps/web/tests/` (they are API integration tests). Playwright config exists ([playwright.config.ts](../../../apps/web/playwright.config.ts)) — investigate coverage there. Treat the redirect decision tree as effectively guarded by production traffic, and say so when proposing changes to it.

What juniors usually miss: `ev.waitUntil` everywhere — the response returns *before* the cache write and click record finish. Also that a cache hit means the DB row is never read, so **anything the redirect needs must be in the cached shape** (`formatRedisLink`).

What seniors notice: the guard order forms invariants (no click tracking before password success — comment at [#L193-L196](../../../apps/web/lib/middleware/link.ts#L193-L196)); the 5-second LRU is a deliberate stampede-breaker, not a real cache; every targeting branch duplicates the recordClick call — a refactor magnet with high blast radius.

Interview angle: "Design a URL shortener" — you can answer with real numbers: 3 cache tiers, 24h Redis TTL, 5s in-process TTL, fire-and-forget analytics, 302-not-301 so analytics aren't bypassed by browser caching (except `_root` which is 301).

Drill: without re-reading, list the guard branches in order, then verify. Then answer: a link is edited — which caches go stale, and what invalidates each? (Check [cache.ts#L50-L64](../../../apps/web/lib/api/links/cache.ts#L50-L64).)

Self-grade — Basic: names middleware + Redis. Solid: names all 3 tiers with TTLs and the guard order. Strong: explains why LRU staleness is bounded at 5s, what `revalidateTag` is invalidating, and the 301/302 tradeoff.

---

## Flow 2: Create a link (`POST /api/links`)

Why this flow matters: the canonical write path — auth, validation, plan gating, DB write, cache priming, and webhook, all in one readable stack.

Open these files first:
- [apps/web/app/api/links/route.ts#L48-L110](../../../apps/web/app/api/links/route.ts#L48-L110) — the route.
- [apps/web/lib/api/links/process-link.ts#L24-L57](../../../apps/web/lib/api/links/process-link.ts#L24-L57) — validation contract.
- [apps/web/lib/api/links/create-link.ts](../../../apps/web/lib/api/links/create-link.ts) — the write + fan-out.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | withWorkspace | [workspace.ts#L58-L80](../../../apps/web/lib/auth/workspace.ts#L58-L80) | authn (token or session), rate limit, tenant resolution, `requiredPermissions: ["links.write"]` | `{workspace, session, token}` | see Flow 5 |
| 2 | Usage gate | [route.ts#L51](../../../apps/web/app/api/links/route.ts#L51) | `throwIfLinksUsageExceeded(workspace)` | — | quota enforcement depends on the async usage counters (Flow 3) |
| 3 | Schema parse | [route.ts#L54-L56](../../../apps/web/app/api/links/route.ts#L54-L56) | `createLinkBodySchemaAsync.parseAsync` (Zod) | `NewLinkProps` | Zod errors become 422 via [errors.ts#L100-L105](../../../apps/web/lib/api/errors.ts#L100-L105) |
| 4 | Anonymous path | [route.ts#L58-L69](../../../apps/web/app/api/links/route.ts#L58-L69) | no session → 10 links/day per IP via Upstash ratelimit | — | header `x-forwarded-for` trust |
| 5 | Business validation | [process-link.ts#L85-L120](../../../apps/web/lib/api/links/process-link.ts#L85-L120) | URL validity, UTM merge, plan-gated features (e.g. root-domain redirect needs Pro) | returns `{link,error,code}` union — no throw | error-as-value keeps bulk flows composable |
| 6 | DB write | [create-link.ts#L55-L138](../../../apps/web/lib/api/links/create-link.ts#L55-L138) | `prisma.link.create` wrapped in `withPrismaRetry`; tags/webhooks/dashboard as nested writes; `shortLink` unique | full `Link` row | uniqueness race on `(domain,key)` surfaces as P2002 |
| 7 | Async fan-out | [create-link.ts#L142-L228](../../../apps/web/lib/api/links/create-link.ts#L142-L228) | `waitUntil` + `Promise.allSettled`: Redis cache set, Tinybird `recordLink`, R2 image upload + follow-up `prisma.link.update`, QStash delayed self-delete for anonymous links (30 min), workspace usage event, webhook trigger propagation, A/B test completion schedule | — | any of these can silently fail — settled, not joined |
| 8 | Webhook | [route.ts#L87-L95](../../../apps/web/app/api/links/route.ts#L87-L95) | `sendWorkspaceWebhook({trigger: "link.created", ...})` inside `waitUntil` | `linkEventSchema` | outbound contract is Zod-parsed *before* send — schema drift fails loudly |

Validation and authorization: three layers, in order — **shape** (Zod, step 3), **business/plan** (processLink, step 5), **tenant/permission** (withWorkspace, step 1). Folder-level access is checked inside processLink via `verifyFolderAccess` (import at [process-link.ts#L2](../../../apps/web/lib/api/links/process-link.ts#L2); call site later in the file — labeled inferred).

Persistence and side effects: MySQL (link + join rows), Redis (link cache), Tinybird (link metadata for analytics joins), R2 (OG images), QStash (delayed job), webhooks.

Tests that cover it: [tests/links/create-link.test.ts](../../../apps/web/tests/links/create-link.test.ts) — real HTTP against a deployed instance, incl. the anonymous path with `dub-anonymous-link-creation` header ([#L22-L48](../../../apps/web/tests/links/create-link.test.ts#L22-L48)); error cases in [create-link-error.test.ts](../../../apps/web/tests/links/create-link-error.test.ts).

What juniors usually miss: `processLink` returns errors as values; only the route converts them to `DubApiError`. Also the image dance: the row is created with `image: null`, uploaded to R2 async, then updated — and the API response *optimistically* returns the final URL ([create-link.ts#L230-L237](../../../apps/web/lib/api/links/create-link.ts#L230-L237)).

What seniors notice: there is **no transaction** spanning the link row and its side effects — consistency between MySQL, Redis, and Tinybird is eventual and repair-less (no outbox). The 100ms `createdAt` increments to preserve tag order ([#L91](../../../apps/web/lib/api/links/create-link.ts#L91)) are a smell worth discussing, not copying.

Interview angle: "How do you keep a cache consistent with the database?" — real answer: write-through on create ([#L149-L155](../../../apps/web/lib/api/links/create-link.ts#L149-L155)), 24h TTL as backstop, and known failure mode (allSettled swallow) you'd fix with an outbox.

Drill: diagram which stores know about a link 0ms, 500ms, and 24h after creation, assuming the Tinybird call failed.

Self-grade — Basic: names route → processLink → createLink. Solid: names all three validation layers and the fan-out list. Strong: identifies the missing-transaction/outbox gap and proposes a repair strategy with its cost.

---

## Flow 3: Click recording (async write fan-out)

Why this flow matters: this is where the repo teaches reliability vocabulary — deduplication, fire-and-forget, partial failure, fallback writes, privacy filtering — in one 360-line file.

Open these files first:
- [apps/web/lib/tinybird/record-click.ts](../../../apps/web/lib/tinybird/record-click.ts) — everything.
- [apps/web/lib/api/links/record-click-cache.ts](../../../apps/web/lib/api/links/record-click-cache.ts) — the 1-hour dedup cache.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Guards | [record-click.ts#L61-L86](../../../apps/web/lib/tinybird/record-click.ts#L61-L86) | skip if no clickId, `dub-no-track`, or bot UA | — | bot detection is heuristic |
| 2 | Dedup | [#L88-L108](../../../apps/web/lib/tinybird/record-click.ts#L88-L108) | identity hash (IP+UA based) checked against Redis; 1 click/hour per (domain,key,identity); **Redis error → drop the click** (protect TB/MySQL over data completeness) | `identityHash` | deliberate availability-over-completeness call — comment at L105 |
| 3 | Enrich | [#L110-L169](../../../apps/web/lib/tinybird/record-click.ts#L110-L169) | geo from Vercel headers, UA parse, QR detection, **EU IPs redacted to empty string** (GDPR) | flat `clickData` (snake_case for Tinybird) | region comes from a header, not `geolocation()` — comment L116-L117 |
| 4 | clickId cache | [#L171-L175](../../../apps/web/lib/tinybird/record-click.ts#L171-L175) | if conversion-relevant, cache full clickData in Redis 5 min — because Tinybird ingestion isn't immediately readable | `clickIdCache:<id>` | conversion within 5 min window depends on this |
| 5 | Fan-out | [#L177-L230](../../../apps/web/lib/tinybird/record-click.ts#L177-L230) | `waitUntil(Promise.allSettled([...]))`: Tinybird HTTP ingest (`fetchWithRetry`), dedup-cache set, raw SQL `UPDATE Link SET clicks = clicks + 1`, workspace usage via **Redis stream with direct-SQL fallback**, partner activity stream ditto | — | counters can drift from Tinybird truth; stream consumer lag |
| 6 | Failure logging | [#L233-L262](../../../apps/web/lib/tinybird/record-click.ts#L233-L262) | rejected promises mapped to named operations and console.error'd | — | observability = logs only; no retry/DLQ here |
| 7 | Webhooks | [#L264-L287](../../../apps/web/lib/tinybird/record-click.ts#L264-L287) + [#L294-L359](../../../apps/web/lib/tinybird/record-click.ts#L294-L359) | if link has webhooks and workspace under usage limit: load webhook configs from cache (**TODO: no DB fallback**, L305-307), raw SQL join for link+tags, dispatch via QStash | — | webhook silently skipped on cache miss — real risk, see [risk register](../09-reference/risk-register.md) |

Validation and authorization: none — inputs come from the middleware which already resolved the link. Privacy filtering (EU IP) is the notable "validation."

Persistence and side effects: Tinybird (source of truth for events), MySQL (denormalized counters — read models for the dashboard), Redis (dedup + clickId cache + streams), QStash (webhooks).

Tests that cover it: no direct test found for `recordClick` in `apps/web/tests/` (integration tests can't easily observe fire-and-forget writes). The `/api/track/*` endpoints have tests under [tests/tracks](../../../apps/web/tests/tracks) that exercise adjacent paths.

What juniors usually miss: raw SQL via the PlanetScale driver ([#L196-L199](../../../apps/web/lib/tinybird/record-click.ts#L196-L199)) instead of Prisma — the comment explains it's about connection pooling on the hot path. Also that `UPDATE ... clicks = clicks + 1` is atomic *in the database*, which is why there's no read-modify-write race.

What seniors notice: dual-write architecture (Tinybird events + MySQL counters) with no reconciliation job visible; stream-publish-with-SQL-fallback ([#L202-L214](../../../apps/web/lib/tinybird/record-click.ts#L202-L214)) is a graceful-degradation pattern worth stealing; the usage-limit check for webhooks reads `Project` on every clicked-webhook link — a hot-path DB read.

Interview angle: "How would you count clicks at scale?" and "What's eventual consistency, concretely?" — this file *is* the answer.

Drill: write the failure matrix — for each of the 5 settled promises, what user-visible thing breaks if it rejects for an hour?

Self-grade — Basic: knows clicks go to Tinybird. Solid: explains dedup, counters vs events, allSettled. Strong: articulates the drift/reconciliation gap and the webhook cache-miss TODO as prioritized risks.

---

## Flow 4: Analytics query (`GET /api/analytics?event=clicks&groupBy=timeseries`)

Why this flow matters: read-side of the analytics pipeline; the best place to see **layered authorization** (workspace → program → link → folder) on one endpoint.

Open these files first:
- [apps/web/app/api/analytics/route.ts](../../../apps/web/app/api/analytics/route.ts) — full route.
- [apps/web/lib/analytics/get-analytics.ts](../../../apps/web/lib/analytics/get-analytics.ts) — Tinybird pipe caller (skimmed only; verify before citing internals).

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | withWorkspace | [route.ts#L20](../../../apps/web/app/api/analytics/route.ts#L20) + [#L135-L137](../../../apps/web/app/api/analytics/route.ts#L135-L137) | `requiredPermissions: ["analytics.read"]`; analytics endpoints get a stricter per-second rate limit ([workspace.ts#L171-L173](../../../apps/web/lib/auth/workspace.ts#L171-L173), [#L30-L33](../../../apps/web/lib/auth/workspace.ts#L25-L34)) | — | — |
| 2 | Usage gate | [route.ts#L22](../../../apps/web/app/api/analytics/route.ts#L22) | `throwIfClicksUsageExceeded` | — | free-tier abuse control |
| 3 | Back-compat | [route.ts#L24-L50](../../../apps/web/app/api/analytics/route.ts#L24-L50) | old `/analytics/[endpoint]` params remapped | — | deprecated contracts kept alive deliberately |
| 4 | Program scope | [route.ts#L52-L68](../../../apps/web/app/api/analytics/route.ts#L52-L68) | `programId` must equal the workspace's default program → 403 otherwise | — | cross-tenant probe blocked here |
| 5 | Link scope | [route.ts#L70-L92](../../../apps/web/app/api/analytics/route.ts#L70-L92) | domain/key/externalId resolved via `getLinkOrThrow` **scoped by workspaceId** — the IDOR guard | filter rewritten to link.id | forgetting the workspace scope here would leak other tenants' stats |
| 6 | Folder authz | [route.ts#L94-L101](../../../apps/web/app/api/analytics/route.ts#L94-L101) | `verifyFolderAccess` — folders add intra-workspace permissions | — | link's own folderId is checked even when user didn't pass one (L89-L91) — subtle and correct |
| 7 | Plan gate | [route.ts#L103-L109](../../../apps/web/app/api/analytics/route.ts#L103-L109) | date-range allowed per plan (`assertValidDateRangeForPlan`) | — | monetization boundary in code |
| 8 | Query | [route.ts#L119-L131](../../../apps/web/app/api/analytics/route.ts#L119-L131) | `getAnalytics` → Tinybird pipes; `console.time` around it | JSON | latency observability is literally console.time |

Validation and authorization: the flow's whole point — four nested scopes on one endpoint. Memorize step 5: **resource lookup functions take `workspaceId` as a parameter** so tenant isolation is structural, not remembered per-route.

Persistence and side effects: read-only against Tinybird (+ MySQL for link/folder lookups).

Tests that cover it: `apps/web/tests/analytics/` exists — retrieval tests against the live instance.

What juniors usually miss: authorization is not one check; it composes. What seniors notice: the deprecated-endpoint shims (L24-L50, L111-L117) show how a public API ages — contracts are forever.

Interview angle: "How do you prevent IDOR?" Answer with `getLinkOrThrow({workspaceId, ...})` — scoping at the query layer, not the route layer.

Drill: enumerate every way a request can get a 403 from this route, with line numbers.

Self-grade — Basic: 2 authz layers. Solid: all four + plan gate. Strong: explains why scoping inside lookup helpers beats per-route `if` checks (defense against the *next* engineer's forgetfulness).

---

## Flow 5: The authorization spine (`withWorkspace`)

Why this flow matters: every workspace API route passes through this one higher-order function. Understanding it means you can safely touch any route in the repo.

Open these files first:
- [apps/web/lib/auth/workspace.ts](../../../apps/web/lib/auth/workspace.ts) — the wrapper (528 lines, read it all).
- [apps/web/lib/api/rbac/permissions.ts#L1-L60](../../../apps/web/lib/api/rbac/permissions.ts#L1-L60) — role→permission table.
- [apps/web/lib/api/tokens/scopes.ts](../../../apps/web/lib/api/tokens/scopes.ts) — token scopes (skimmed).

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Parse credentials | [workspace.ts#L105-L120](../../../apps/web/lib/auth/workspace.ts#L105-L120) | `Bearer` header → apiKey; `dub_` prefix marks a *restricted* (workspace-scoped) token | string | helpful error message for missing `Bearer ` |
| 2 | Tenant hint | [#L122-L169](../../../apps/web/lib/auth/workspace.ts#L122-L169) | workspace id/slug from params or query; `ws_` prefix normalized; anonymous link-creation carve-out (L134-L146) | id or slug | the carve-out bypasses *everything* — small but sharp surface |
| 3 | Token auth | [#L175-L235](../../../apps/web/lib/auth/workspace.ts#L175-L235) | SHA-hash the key, Redis token cache → Prisma `restrictedToken`/`token` lookup, expiry check, cache set | `TokenCacheItem` | secrets stored hashed (good); cache poisoning window on revoke — investigate token-cache invalidation on delete |
| 4 | Rate limit | [#L237-L265](../../../apps/web/lib/auth/workspace.ts#L237-L265) (tokens), [#L320-L339](../../../apps/web/lib/auth/workspace.ts#L320-L339) (sessions) | plan-based limits; analytics endpoints per-second; headers surfaced to caller | 429 on exceed | limits keyed by hashed token / user id |
| 5 | lastUsed bookkeeping | [#L272-L300](../../../apps/web/lib/auth/workspace.ts#L272-L300) | `waitUntil` + a 1/min ratelimit gate so the UPDATE runs at most once a minute | — | clever: rate limiter as write-throttle |
| 6 | Tenant membership | [#L342-L402](../../../apps/web/lib/auth/workspace.ts#L342-L402) | fetch workspace **with this user's membership row**; not a member → pending-invite check → `not_found` (not 403!) | `WorkspaceWithUsers` | returning 404 for unauthorized hides tenant existence — deliberate |
| 7 | Permissions | [#L404-L439](../../../apps/web/lib/auth/workspace.ts#L404-L439) | role → permissions; restricted-token scopes **intersected** with role permissions; `requiredPermissions` / `requiredRoles` enforced | `PermissionAction[]` | intersection means a token can never exceed its user |
| 8 | Plan/flag gates | [#L441-L473](../../../apps/web/lib/auth/workspace.ts#L441-L473) | feature flags via Edge Config; plan allowlist; free-plan analytics-API block | 403 | monetization enforced at the boundary |
| 9 | Handler + logging | [#L475-L524](../../../apps/web/lib/auth/workspace.ts#L475-L524) | handler runs; success *and* error paths `captureRequestLog` via `waitUntil`; all errors → `handleAndReturnErrorResponse` | `Response` | error path clears `workspace` when lookup failed (L362-L364) to avoid mis-attributed logs |

Validation and authorization: this *is* the authorization layer. Note the ordering: authenticate → rate-limit → resolve tenant → membership → role/scope intersection → plan. Each later check assumes the earlier ones.

Persistence and side effects: Redis (token cache, rate limits), MySQL (token, workspace, invite), Axiom (request logs).

Tests that cover it: indirectly by every integration test (they all pass a real token — [tests/utils/integration.ts#L26-L31](../../../apps/web/tests/utils/integration.ts#L26-L31)).

What juniors usually miss: two credential worlds (session cookie vs Bearer token) converge into one `session` object at [#L302-L309](../../../apps/web/lib/auth/workspace.ts#L302-L309), so handlers never care which was used.

What seniors notice: scope-permission **intersection** (L413-L418) is the key invariant; 404-for-unauthorized as an anti-enumeration choice; machine users force-promoted to owner (L404-L408) — a sharp edge to know about before touching RBAC.

Interview angle: "Design API keys for a multi-tenant SaaS" — hashed storage, workspace-scoped tokens, scope∩role, plan-based rate limits, token cache with TTL. You have a production reference implementation.

Drill: a request arrives with a valid `dub_` token whose user was removed from the workspace yesterday. Walk the code — where exactly does it fail? (Follow step 6.)

Self-grade — Basic: "it checks the API key and workspace." Solid: order of gates + both credential modes. Strong: names the intersection invariant, the 404 choice, and the token-revocation cache-invalidation question.

---

## Flow 6: Outbound webhooks via QStash (background delivery)

Why this flow matters: the repo's cleanest example of **at-least-once delivery** delegated to a queue, with signatures, callbacks, and failure handling.

Open these files first:
- [apps/web/lib/webhook/qstash.ts](../../../apps/web/lib/webhook/qstash.ts) — publish path.
- [apps/web/lib/webhook/publish.ts](../../../apps/web/lib/webhook/publish.ts) — `sendWorkspaceWebhook` entry point.
- [apps/web/app/api/webhooks/callback/route.ts](../../../apps/web/app/api/webhooks/callback/route.ts) — delivery-result handling (listed, not fully read — verify before citing).

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Trigger | e.g. [app/api/links/route.ts#L87-L95](../../../apps/web/app/api/links/route.ts#L87-L95) | domain event (`link.created`) → `sendWorkspaceWebhook` inside `waitUntil` | Zod-parsed event | event schema enforced at the producer |
| 2 | Fan to endpoints | [qstash.ts#L16-L36](../../../apps/web/lib/webhook/qstash.ts#L16-L36) | one QStash publish per registered webhook | `webhookPayloadSchema` | — |
| 3 | Sign + publish | [qstash.ts#L39-L106](../../../apps/web/lib/webhook/qstash.ts#L39-L106) | HMAC `Dub-Signature` from per-webhook secret; success/failure **callback URLs** back into the app; receiver-specific transforms (Slack/Segment, L109-L124) | signed JSON | `TODO: Add deduplicationId` (L69-L71) — duplicate deliveries possible on retry |
| 4 | Delivery | QStash (external) | retries with backoff, then failure callback | — | consumer must verify signature & be idempotent |
| 5 | Result callback | [app/api/webhooks/callback/route.ts](../../../apps/web/app/api/webhooks/callback/route.ts) | records webhook event, handles repeated failures (see [lib/webhook/failure.ts](../../../apps/web/lib/webhook/failure.ts) — auto-disable logic; verify details before citing) | — | failure escalation policy |

Validation and authorization: producer-side Zod parse (step 1); consumer-side, receivers verify `Dub-Signature`. Inbound QStash calls to cron/callback routes are verified with [verify-qstash.ts](../../../apps/web/lib/cron/verify-qstash.ts) — note verification is **skipped off-Vercel** ([#L18-L21](../../../apps/web/lib/cron/verify-qstash.ts#L18-L21)).

Persistence and side effects: QStash queue; webhook event log to Tinybird ([lib/tinybird/record-webhook-event.ts](../../../apps/web/lib/tinybird/record-webhook-event.ts)); webhook disable-on-failure updates MySQL.

Tests that cover it: `NODE_ENV === "test"` adds a 5s delivery delay ([qstash.ts#L88](../../../apps/web/lib/webhook/qstash.ts#L88)) — a test hook baked into prod code; webhook tests live under `apps/web/tests/webhooks/` (directory not exhaustively read).

What juniors usually miss: delivery is *not* an HTTP call from the request handler — it's queued, so "webhook sent" in code means "accepted by QStash."

What seniors notice: the missing deduplicationId TODO is an idempotency gap the *consumers* currently pay for; callback-URL-into-own-API is a neat pattern but makes the app a consumer of its own public surface.

Interview angle: "How do you deliver webhooks reliably?" — signatures, retries via queue, failure callbacks, auto-disable, and the honest gap (no dedup id yet).

Drill: as a webhook consumer, what three things must you implement to consume Dub webhooks safely? (Verify signature; respond 2xx fast/process async; dedupe by event id.)

Self-grade — Basic: "QStash sends them." Solid: signature + callbacks + retry story. Strong: identifies the idempotency gap and states what deduplicationId would change.

---

## Flow 7: Dashboard links list (client → API → SWR cache)

Why this flow matters: the repo's standard client data-fetching pattern — SWR keyed by URL, filters serialized into the querystring, one hook per resource.

Open these files first:
- [apps/web/lib/swr/use-links.ts#L11-L50](../../../apps/web/lib/swr/use-links.ts#L11-L50) — the hook.
- [apps/web/lib/swr/use-workspace.ts](../../../apps/web/lib/swr/use-workspace.ts) — workspace context every hook leans on (not fully read).
- [apps/web/app/api/links/route.ts#L20-L45](../../../apps/web/app/api/links/route.ts#L20-L45) — the GET it calls.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Component | `apps/web/ui/links/*` | renders list, reads filters from URL via `useRouterStuff` | — | URL is the state container |
| 2 | Hook | [use-links.ts#L11-L50](../../../apps/web/lib/swr/use-links.ts#L11-L50) | SWR key = `/api/links?workspaceId=...&<filters>`; skipped until `workspaceId` exists; mega-workspaces force `showArchived=false` | `ExpandedLinkProps[]` | key = cache identity — any param change is a new cache entry |
| 3 | API | [route.ts#L20-L45](../../../apps/web/app/api/links/route.ts#L20-L45) | Zod-parse filters, `validateLinksQueryFilters` (folder access), fuzzy vs exact search by workspace size | JSON array | search mode flips at `MEGA_WORKSPACE_LINKS_LIMIT` — perf guard |
| 4 | Mutation | [lib/swr/mutate.ts](../../../apps/web/lib/swr/mutate.ts) | after create/update, SWR keys revalidated (prefix-based mutation helpers) | — | forgetting to mutate = stale list until refocus |

Validation and authorization: same `withWorkspace` spine; folder filtering validated server-side (never trust client filter params).

Persistence and side effects: read-only; client cache in SWR.

Tests that cover it: [tests/links/list-links.test.ts](../../../apps/web/tests/links/list-links.test.ts) covers the API; the hook itself has no tests (typical — the contract is tested at the API).

What juniors usually miss: SWR's key-as-identity. Two components calling `useLinks({page: 1})` share one request; `{page: 2}` is a different cache row. What seniors notice: server-driven guardrails for big tenants (search mode, archived filter) — the client adapts to data scale signals from `useWorkspace`.

Interview angle: "How does SWR/React Query caching work?" — answer with the key construction in this hook, stale-while-revalidate defaults, and mutate-on-write.

Drill: you archive a link and the list doesn't update until tab refocus. Which line was forgotten, and what key must be mutated?

Self-grade — Basic: "SWR fetches /api/links." Solid: key construction + conditional fetch + mutate. Strong: explains request dedup, the mega-workspace guardrails, and when you'd switch to server components instead.
