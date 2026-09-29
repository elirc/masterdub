# Annotation Drills

Eight real excerpts. For each: open the anchor, and annotate — **inputs, outputs, dependencies, invariants, side effects, failure modes** — before reading the "what to find" notes. Write your annotations down; grading criteria at the bottom apply to every drill.

---

## Drill 1: The cache read path

Open [apps/web/lib/api/links/cache.ts#L66-L100](../../../apps/web/lib/api/links/cache.ts#L66-L100) (`LinkCache.get`).

Annotate, then check yourself:
- **Inputs:** `{domain, key}` — but is `key` already normalized? (Who is responsible — caller or cache? See [link.ts#L48-L52](../../../apps/web/lib/middleware/link.ts#L48-L52). The answer is "caller," which makes normalization an *implicit contract*.)
- **Invariant:** LRU entry ⊆ Redis entry ⊆ DB truth, each with bounded staleness (5s / 24h).
- **Side effects:** a read *writes* — LRU repopulation on Redis hit; that's cache-aside.
- **Failure modes:** Redis timeout falls through to Vercel cache (L97-L100) — what's the staleness bound in that degraded mode? (5 min.)
- **Trap:** the comment at L67-L70 explains why the raw string key is used instead of `_createKey` — what bug does that prevent?

## Drill 2: The guard ladder

Open [apps/web/lib/middleware/link.ts#L175-L252](../../../apps/web/lib/middleware/link.ts#L175-L252) (inspect → password → banned → disabled → expired).

- **Invariant to find:** no click is recorded before the password gate passes (comment L193-L196). Which guards return *rewrites* (URL unchanged in browser) vs *redirects*, and why does that distinction matter for SEO and for the user's back button?
- **Dependency:** the password check re-fetches the link from the DB (L197) even though a cached copy exists — annotate why (the cached `RedisLinkProps` stores password presence, but the comparison uses the DB value; also keeps the secret out of… does it? investigate whether `password` is in the Redis shape).
- **Failure mode:** banned check compares `workspaceId === LEGAL_WORKSPACE_ID` (L213) — a sentinel-value design. What breaks if a migration renumbers workspaces?

## Drill 3: Token authentication block

Open [apps/web/lib/auth/workspace.ts#L175-L235](../../../apps/web/lib/auth/workspace.ts#L175-L235).

- **Inputs:** raw `apiKey` string; **outputs:** a `token` record with user; **dependency:** Redis token cache, then Prisma.
- **Invariant:** only the *hash* of the key ever touches storage or cache keys.
- **Side effects:** cache set inside `waitUntil` (L228-L235) — annotate the race: two concurrent first-requests both miss and both set. Harmless? Why?
- **Failure mode to find:** expiry is checked (L221-L226) *after* cache read — is `expires` part of the cached item? If a token expires while cached, which line catches it? (L221 runs on cached tokens too — verify by reading the flow order.)

## Drill 4: The anonymous carve-out

Open [apps/web/lib/auth/workspace.ts#L128-L161](../../../apps/web/lib/auth/workspace.ts#L128-L161).

- **Invariant violated deliberately:** the handler contract (`session`, `workspace` non-null) — hence `@ts-expect-error` at L140. Annotate every downstream assumption this breaks and where the route handles it ([links route L58-L69](../../../apps/web/app/api/links/route.ts#L58-L69), `if (!session)`).
- **Security boundary:** what limits abuse? (IP rate limit 10/day; 30-min self-destruct via QStash at [create-link.ts#L199-L208](../../../apps/web/lib/api/links/create-link.ts#L199-L208).) Annotate: are those two controls in the same file as the carve-out? What does that distance cost?

## Drill 5: Nested-write link creation

Open [apps/web/lib/api/links/create-link.ts#L55-L138](../../../apps/web/lib/api/links/create-link.ts#L55-L138).

- **Inputs:** `ProcessedLinkProps` (already validated — annotate what "already" means and who guaranteed it).
- **Side effects:** one SQL transaction? Prisma nested writes (tags, webhooks, dashboard) are one `create` → yes, atomic *within MySQL*. The `waitUntil` block after is not. Draw the atomicity boundary.
- **Invariant:** `shortLink` (L61) must equal `https://{domain}/{key}` for the row to be findable by the unique column — two representations of one fact.
- **Failure mode:** duplicate `(domain,key)` → Prisma P2002 → generic 422 via [route.ts#L100-L105](../../../apps/web/app/api/links/route.ts#L100-L105). Annotate what the user sees vs what the actual conflict was — is the message good enough?
- **Oddity to explain:** `createdAt` skewed by `idx * 100` ms for tags (L91, L103) — the DB has no order column, so insertion time *is* the order. Annotate the failure mode (clock as sequence).

## Drill 6: EU IP redaction and identity

Open [apps/web/lib/tinybird/record-click.ts#L110-L169](../../../apps/web/lib/tinybird/record-click.ts#L110-L169).

- **Inputs:** request headers (Vercel geo headers — annotate the trust assumption: who can spoof these off-Vercel?).
- **Invariant:** EU visitor IPs are never stored (L143-L145) — but `identityHash` (L88) is derived from IP earlier. Annotate whether the hash constitutes personal data under the same policy (this is a *question to raise*, not a bug claim — see [risk register](../../09-reference/risk-register.md)... investigate `get-identity-hash.ts` first: it may salt/rotate).
- **Output:** flat snake_case `clickData` — annotate why snake_case (Tinybird column names — the wire format leaks inward, a boundary observation).

## Drill 7: SWR key construction

Open [apps/web/lib/swr/use-links.ts#L11-L50](../../../apps/web/lib/swr/use-links.ts#L11-L50).

- **Inputs:** filter opts + ambient workspace; **output:** links array, but also *loading/error states* — annotate all three consumer-visible states.
- **Invariant:** SWR key string === cache identity === request identity. Annotate what happens when `workspaceId` is briefly `undefined` on first render (conditional fetching — the `null` key).
- **Dependency:** `useWorkspace()` — a hook calling a hook; annotate the render cascade when workspace loads.
- **Failure mode:** two components with slightly different `opts` object shapes → different keys → duplicate requests for near-identical data.

## Drill 8: QStash publish with callbacks

Open [apps/web/lib/webhook/qstash.ts#L39-L106](../../../apps/web/lib/webhook/qstash.ts#L39-L106).

- **Inputs:** webhook (url+secret) + already-validated payload; **outputs:** QStash messageId — annotate that "sent" here means *enqueued*, and where actual delivery outcome lands (callback URLs, L52-L63).
- **Invariant:** every payload is signed with the per-webhook secret before leaving (L67).
- **Side effects:** none locally! The function is pure orchestration — annotate why that makes it easy to test (and check whether it *is* tested).
- **Failure modes:** `!response.messageId` is only console.error'd (L91-L93) — the event is silently lost. Annotate severity: what would you do instead? Also the `TODO: deduplicationId` (L69-L71) — annotate which party currently absorbs duplicate deliveries.

---

## Self-grading rubric (apply to each drill)

- **Basic:** you identified inputs/outputs and the obvious happy path. You can say what the code does.
- **Solid:** you named at least one invariant *not stated in comments*, all external dependencies, and every side effect including cache writes and `waitUntil` work. You can say what the code assumes.
- **Strong:** you identified who *else* depends on each invariant (call sites, other files), at least one realistic failure mode with its blast radius, and one question you'd raise in review rather than assert as a bug. You can say what the code costs.

If you scored Basic on drills 2, 3, or 5 — reread [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md) Flows 1, 5, 2 respectively before continuing.
