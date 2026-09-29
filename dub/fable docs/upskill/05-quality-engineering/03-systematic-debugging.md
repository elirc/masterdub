# Systematic Debugging

The method, then five repo-specific scenarios. Interviews increasingly include a live debugging round; each scenario ends with how to narrate it aloud.

## The method

1. **Reproduce** — a bug you can't reproduce is a rumor. Pin inputs, environment, and frequency.
2. **Narrow** — binary-search the pipeline. In this repo the pipeline is almost always: client → middleware/wrapper → route → domain function → store(s) → async fan-out. Ask "is the bad data already bad *here*?" at each boundary.
3. **Hypothesize** — one falsifiable cause at a time.
4. **Test cheaply** — a log line, a curl, a Redis `GET`, a SQL query — before any code change.
5. **Fix the root cause** — not the symptom; check for siblings of the same bug class.
6. **Add regression coverage** — in this repo that usually means an integration test under `apps/web/tests/`.

The repo's actual probes: `console.log` cache HIT/MISS lines already in [cache.ts#L76-L94](../../../apps/web/lib/api/links/cache.ts#L76-L94); Axiom structured logs ([lib/axiom/server](../../../apps/web/lib/axiom/server.ts)); Prisma Studio (`pnpm prisma:studio`, runs with `dev`); direct Redis inspection via Upstash console; Tinybird SQL console; QStash dashboard for queued/failed messages; browser devtools + the `+`-suffix inspect mode on any link ([link.ts#L54-L58](../../../apps/web/lib/middleware/link.ts#L54-L58)) — a built-in prod debugging feature.

---

## Scenario 1: "I edited my link's destination but it still redirects to the old URL"

Reproduction: edit a link via dashboard; click the short link within seconds from another machine — old destination.
First question: is the bug on the write path (cache not updated) or the read path (stale tier)?
Narrowing path:
1. Check MySQL (Prisma Studio): is `Link.url` updated? Yes → write path committed; it's a cache problem.
2. Check Redis (`GET linkcache:<domain>:<key>`): updated? If yes → the stale read is the in-process LRU ([cache.ts#L18-L21](../../../apps/web/lib/api/links/cache.ts#L18-L21)) on *another instance* — bounded at 5s; if the symptom persists for minutes, that hypothesis is falsified.
3. If Redis is stale → did the update path call `linkCache.set`? Find the equivalent of the create-path cache write in [update-link.ts](../../../apps/web/lib/api/links/update-link.ts) and check for a changed cache key: **did the domain or key change during the edit?** Old cache entry `linkcache:old-domain:old-key` may not have been deleted.
Useful probes: the `[LRU Cache HIT]` log lines; Redis TTL of the entry (24h fresh vs old).
Likely root causes: key-change edit leaves the old cache entry live; or Redis `set` failed silently inside `waitUntil`.
Regression test to add: integration test — update a link's key, assert old short link 404s and new one redirects.
Senior lesson: cache invalidation bugs are usually *key-identity* bugs, not TTL bugs.
Interview version: narrate the tiers in order of staleness bound (5s / 24h / 5m) and how each observation eliminates a tier — that ordering *is* the demonstration of method.

## Scenario 2: "Click counts on the dashboard don't match Tinybird analytics"

Reproduction: `Link.clicks` = 1,204; analytics timeseries sums to 1,297.
First question: which number is supposed to be authoritative? (Events in Tinybird — counters are read models; see [pattern 12](../03-architecture-and-patterns/05-pattern-catalog.md).)
Narrowing path:
1. Check whether the gap grows or is fixed-size. Fixed → historical incident (e.g. Redis stream fallback double-write); growing → live divergence.
2. Read the fan-out ([record-click.ts#L177-L230](../../../apps/web/lib/tinybird/record-click.ts#L177-L230)): the Tinybird ingest and the SQL `clicks + 1` are *independent* settled promises — either can fail alone. Search logs for `[Record click] - Rejected promises`.
3. Check dedup asymmetry: is anything writing to Tinybird that skips the counter (e.g. `/api/track/click` with `skipRatelimit` — [record-click.ts#L92](../../../apps/web/lib/tinybird/record-click.ts#L92))? Compare `trigger` values in Tinybird rows.
Useful probes: Tinybird SQL grouped by `trigger`; error-log counts per operation name.
Likely root causes: partial fan-out failures over time; different dedup rules across entry points.
Regression test to add: hard — instead propose a **reconciliation cron** that samples links and reports drift (see mid-level tickets).
Senior lesson: dual-writes *will* drift; the question is whether you can measure and repair, not whether you can prevent.
Interview version: "First I establish which store is the source of truth; then I look for the code path where the two writes can diverge; then I quantify the drift before fixing anything."

## Scenario 3: "Webhook stopped firing for link clicks"

Reproduction: customer reports no `link.clicked` deliveries since yesterday; other events fine.
First question: is the event not being *produced* (our side) or not being *delivered* (QStash → them)?
Narrowing path:
1. QStash dashboard: messages published for this webhook id? If yes with failures → delivery problem; check the failure callback handling and whether the webhook got **auto-disabled** ([lib/webhook/failure.ts](../../../apps/web/lib/webhook/failure.ts); `disabledAt` filter at [record-click.ts#L311-L318](../../../apps/web/lib/tinybird/record-click.ts#L311-L318)).
2. If nothing published: the click path loads webhook configs **from cache only** — [record-click.ts#L303-L309](../../../apps/web/lib/tinybird/record-click.ts#L303-L309) with an explicit `TODO: Should we look them up in the database?` A webhook cache miss (eviction, Redis flush) silently skips delivery.
3. Also check the usage-limit gate ([#L280-L286](../../../apps/web/lib/tinybird/record-click.ts#L280-L286)): workspace over its clicks quota also silently stops webhooks.
Useful probes: Redis `MGET` the webhook cache keys; `Project.usage` vs `usageLimit` row.
Likely root causes: auto-disable after their endpoint failed; webhook cache miss; usage limit exceeded.
Regression test to add: unit-level test of the filter logic in `sendLinkClickWebhooks` with a cache-miss stub; product fix = DB fallback for the cache (a ready-made contribution).
Senior lesson: silent-skip paths (cache miss, quota gates) need *at minimum* a counter metric — "no error" is not "working."
Interview version: model it as producer/transport/consumer, eliminate transport first (queue dashboards make that cheap), then walk the producer's guard conditions in code order.

## Scenario 4: "API returns 404 'Workspace not found' but the workspace exists"

Reproduction: customer's script gets 404 on `GET /api/links?workspaceId=ws_xxx` with a valid-looking key.
First question: which of the several deliberate-404 branches are we in? ([workspace.ts](../../../apps/web/lib/auth/workspace.ts) returns 404 for: workspace truly missing L361-L370; **user not a member** L386-L389; workspace-id mismatch with a restricted token's own project L268-L269 leading to a different workspace lookup.)
Narrowing path:
1. Decode the credential type: does the key start with `dub_`? Then it's a restricted token and `workspaceId` comes from the *token*, not the query param (L268-L270) — if they pass a different workspace's id in the query it's ignored… verify: actually the token's projectId overrides. So a token from workspace A + `workspaceId=B` → looks up A. If the user was removed from A → 404.
2. Check membership: `ProjectUsers` row for (user, workspace)?
3. Check invite state: pending invite produces `invite_pending`, expired produces `invite_expired` (L391-L401) — different codes, useful signal.
Useful probes: the error `code` field in the response body ([errors.ts](../../../apps/web/lib/api/errors.ts) contract) — clients often read only the status.
Likely root causes: token belongs to a different workspace than intended; user removed from workspace (tokens don't die with membership — *verify* whether removal revokes restricted tokens; if not, that's a finding for the risk register).
Regression test to add: integration test asserting the exact error code for a foreign-workspace token.
Senior lesson: deliberate 404-for-403 anti-enumeration makes support harder — the error *code* taxonomy is what keeps it debuggable.
Interview version: enumerate the distinct branches that produce the same status code, then pick observations that distinguish them — this is "differential diagnosis" and interviewers recognize it instantly.

## Scenario 5: "Anonymous link creation suddenly 401s in production"

Reproduction: the marketing site's "try it" widget fails; authenticated API is fine.
First question: which condition of the carve-out ([workspace.ts#L134-L146](../../../apps/web/lib/auth/workspace.ts#L134-L146)) stopped matching — the header or the pathname?
Narrowing path:
1. The carve-out requires header `dub-anonymous-link-creation` **and** `req.nextUrl.pathname ∈ {"/links", "/api/links"}`. Check what pathname the widget actually hits now (a proxy/rewrite change upstream alters `nextUrl.pathname` — middleware rewrites are upstream of this code).
2. Check the header survives any CDN/proxy (some strip non-standard headers).
3. If both hold, check the IP rate limit (10/day — [links route L58-L69](../../../apps/web/app/api/links/route.ts#L58-L69)): a NAT'd office or a CDN collapsing client IPs into one `x-forwarded-for` exhausts the quota for everyone behind it.
Useful probes: log `req.nextUrl.pathname` at the carve-out; curl with and without the header from outside the CDN.
Likely root causes: infra change rewrote the path; header stripped; shared-IP rate-limit exhaustion (returns 429 not 401 — so a true 401 points at the first two).
Regression test to add: integration test for the anonymous path already exists ([create-link.test.ts#L22-L48](../../../apps/web/tests/links/create-link.test.ts#L22-L48)) — extend with an assertion on the exact status when the header is missing.
Senior lesson: exceptions to auth are coupling bombs — they depend on exact paths and headers that infra teams change without knowing.
Interview version: "the bug is at the boundary between two teams' systems" is a story senior interviewers lean into — narrate how you'd confirm with evidence rather than blame.
