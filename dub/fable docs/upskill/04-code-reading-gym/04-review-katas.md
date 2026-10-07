# Review Katas

Eight fake PRs. For each: read the intent and diff summary, write your review (blocking / important / optional findings + one kind, specific comment), then check the expected findings. The diffs are **fake** but written against real files — open the anchors to review in context.

Review language rule used throughout: comment on the code, offer the reason, propose a path. "This leaks X because Y — could we Z?" beats "wrong."

---

## Kata 1: "Add clickLimit to links"

Author intent: links auto-disable after N clicks.
Fake diff summary: adds `clickLimit Int?` to [link.prisma](../../../packages/prisma/schema/link.prisma); in [record-click.ts](../../../apps/web/lib/tinybird/record-click.ts) after the counter UPDATE, reads the row and sets `disabledAt` if `clicks >= clickLimit`; UI field in the link builder.
Your task: review it.
Expected findings —
Blocking: the redirect path reads the *cached* link ([link.ts#L85](../../../apps/web/lib/middleware/link.ts#L85)); `formatRedisLink` was not updated, so `disabledAt` set by the click path won't be seen until the 24h TTL — feature doesn't work for cached (i.e., all hot) links unless the cache is invalidated on disable.
Important: read-after-update in the fan-out is racy around the limit (a burst can overshoot); acceptable? state it. New field also unused by `processLink` validation (negative values?).
Optional: naming — `clickLimit` vs existing `usageLimit` convention on workspace.
Good review comment example:
> The disable write looks right, but the hot path serves from Redis (`linkCache.get`, link.ts L85) — after this sets `disabledAt` we need `linkCache.set`/expire for the change to take effect before the TTL. Same pattern as delete-link's invalidation; want me to point you at it?

## Kata 2: "Speed up links list by removing folder check"

Author intent: `GET /api/links` is slow for a big customer; profiling shows `validateLinksQueryFilters` adds a query.
Fake diff summary: removes the `validateLinksQueryFilters` call from [links route L24-L28](../../../apps/web/app/api/links/route.ts#L24-L28), reasoning "withWorkspace already checks membership."
Expected findings —
Blocking: folder ACLs are a *second* authorization layer ([lib/folder/permissions.ts](../../../apps/web/lib/folder/permissions.ts)); membership ≠ folder access. This leaks links from restricted folders to all members — a permission regression that tests may not catch (check [folder-link-access.test.ts](../../../apps/web/tests/links/folder-link-access.test.ts) — it exists!).
Important: the perf claim is unmeasured relative to alternatives (cache folder permissions, index).
Optional: none.
Lesson: perf PRs that delete authz are the classic trap; the reviewer's job is to know *why* the "redundant" check exists.

## Kata 3: "Retry webhook publish on failure"

Author intent: QStash publish occasionally fails; wrap `publishWebhookEventToQStash` ([qstash.ts#L39-L106](../../../apps/web/lib/webhook/qstash.ts#L39-L106)) in a 3-attempt retry loop.
Expected findings —
Blocking: without a `deduplicationId` (the TODO at L69-L71), a publish that *succeeded but timed out on response* + retry = duplicate deliveries to customers. Retry must land together with dedup or be idempotent-safe.
Important: retrying inside `waitUntil` extends background time; backoff unbounded? Also `!response.messageId` at L91 isn't a throw — the retry loop as written wouldn't even trigger; the failure mode is a *silent* non-publish.
Optional: metrics on retry counts.
Lesson: "add a retry" is never free; ask "what makes the operation safe to repeat?"

## Kata 4: "Refactor: await the fan-out in createLink for reliability"

Author intent: worried about lost side effects, author changes `waitUntil(...)` to `await (...)` in [create-link.ts#L142](../../../apps/web/lib/api/links/create-link.ts#L142).
Expected findings —
Blocking: adds R2 upload + Tinybird + QStash latency to every link-create API call; timeout risk on the route; the anonymous-widget path becomes user-visibly slow. Reliability gained is marginal (failures still only logged).
Important: if reliability is the real goal, the correct shape is an outbox/queue, not synchronous coupling ([critique #5](../03-architecture-and-patterns/06-architecture-critique.md)).
Optional: could await *just* the cache set (cheap, fixes read-your-writes) — a defensible middle.
Lesson: distinguish "must complete before response" from "must complete eventually" per effect; wholesale await/waitUntil flips are both wrong.

## Kata 5: "Add `GET /api/links/by-external-id/[externalId]`"

Author intent: convenience endpoint.
Fake diff summary: new route calling `prisma.link.findFirst({ where: { externalId: params.externalId } })` wrapped in `withWorkspace` with `requiredPermissions: ["links.read"]`.
Expected findings —
Blocking: `findFirst` by externalId **without projectId** — cross-tenant read (externalIds are only unique per workspace: `@@unique([projectId, externalId])`, [link.prisma#L96](../../../packages/prisma/schema/link.prisma#L96)). Must use the composite key.
Important: duplicate surface — `GET /api/links/info` already supports externalId lookup (check [app/api/links/info](../../../apps/web/app/api/links/info)); new endpoints on a public API are forever.
Optional: response should reuse the existing link response schema.
Lesson: uniqueness scope is authorization scope.

## Kata 6: "Improve dashboard perf with useMemo everywhere"

Fake diff summary: wraps every variable in [link-card.tsx](../../../apps/web/ui/links/link-card.tsx) in `useMemo`/`useCallback`, adds `memo` to five leaf components.
Expected findings —
Important: no measurement (React Profiler) motivating it; `useMemo` on primitives ([#L63-L67](../../../apps/web/ui/links/link-card.tsx#L63-L67) memoizes a Boolean — the existing code already borders on this) costs more than it saves; memo on components receiving fresh object props each render does nothing.
Optional: the real wins are usually list virtualization or fetch shape, not micro-memo.
Blocking: none — it's just churn. Practice writing a review that *declines* a PR kindly and asks for the profile first.
Lesson: performance PRs need a before/after number or they're style PRs.

## Kata 7: "Log full request body for debugging 422s"

Fake diff summary: in [errors.ts fromZodError](../../../apps/web/lib/api/errors.ts#L64-L90), adds `console.log(JSON.stringify(requestBody))` on validation failures.
Expected findings —
Blocking: request bodies contain PII/secrets (customer emails on `/track/lead`, passwords on link creation) → logs become a compliance liability; EU IP redaction elsewhere ([record-click.ts#L143-L145](../../../apps/web/lib/tinybird/record-click.ts#L143-L145)) shows the repo's privacy bar.
Important: the repo already has structured body logging with a dedicated wrapper (`withAxiomBodyLog`, [workspace.ts#L81](../../../apps/web/lib/auth/workspace.ts#L81)) — extend that (with redaction) rather than adding a second path.
Optional: log the Zod issue paths (field names only) — usually enough to debug.
Lesson: observability changes are security changes.

## Kata 8: "Bulk delete links by tag"

Author intent: `DELETE /api/links/bulk?tagId=...`.
Fake diff summary: fetches all link ids with the tag, then `prisma.link.deleteMany`, then loops `linkCache.expire` per link, fires one `link.deleted` webhook *per link* synchronously.
Expected findings —
Blocking: unbounded operation — a tag can have 50k links; needs pagination/chunking and a background job for large sets (compare the existing [bulk-delete-links.ts](../../../apps/web/lib/api/links/bulk-delete-links.ts) and the cron-based domain-deletion workflows under [app/(ee)/api/cron/domains](../../../apps/web/app/(ee)/api/cron/domains)); missing folder-permission check on the victims; webhook loop inside the request = timeout.
Important: cache invalidation should use `mset`/pipeline patterns ([cache.ts#L33-L48](../../../apps/web/lib/api/links/cache.ts#L33-L48)); counters (`Project.totalLinks`, Tinybird `recordLink` deletions) must be updated too — deletion has the same fan-out obligations as creation.
Optional: soft-delete/undo consideration.
Lesson: destructive bulk endpoints are where juniors create incidents; the checklist is bounds, authz-per-item, side-effect parity with single-item paths, and background execution.

---

Self-grading across katas — Basic: caught the blocking issue in ≥4. Solid: caught blocking in ≥6 and correctly *tiered* findings (didn't block on style). Strong: 8/8 blocking finds, plus each review includes one genuine question (not rhetorical) and one alternative path, phrased so the author keeps their dignity. Convert katas 1, 2, and 8 into timed interview practice via [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
