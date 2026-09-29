# Side Effects, Async, and Reliability

## The side-effect map

| Effect | Trigger | Mechanism | Guarantee |
| --- | --- | --- | --- |
| Click event → Tinybird | redirect | `waitUntil` + `fetchWithRetry` ([record-click.ts#L179-L189](../../../apps/web/lib/tinybird/record-click.ts#L179-L189)) | best-effort, retried, lossy on final failure |
| Counter increments (Link, Project, Enrollment) | click | raw SQL / Redis stream + SQL fallback ([#L194-L229](../../../apps/web/lib/tinybird/record-click.ts#L194-L229)) | at-least-once-ish; drift possible |
| Link cache write | create/update | write-through in `waitUntil` ([create-link.ts#L149-L155](../../../apps/web/lib/api/links/create-link.ts#L149-L155)) | self-healing on miss (DB fallback) |
| Outbound webhooks | domain events + clicks | QStash publish, signed, callbacks ([qstash.ts#L39-L106](../../../apps/web/lib/webhook/qstash.ts#L39-L106)) | at-least-once via queue retries; **no dedup id yet** (L69-L71) |
| Emails | invites, limits, digests | `@dub/email` + cron jobs (`send-batch-email`, `trial-emails` under [app/(ee)/api/cron](../../../apps/web/app/(ee)/api/cron)) | queue/cron-driven |
| Image upload → R2 + row update | link create w/ proxy image | two-step: null then update ([create-link.ts#L174-L197](../../../apps/web/lib/api/links/create-link.ts#L174-L197)) | window where row lacks image; response lies optimistically (L230-L237) |
| Delayed self-delete (anonymous links) | create w/o user | QStash `delay: 30*60` ([create-link.ts#L199-L208](../../../apps/web/lib/api/links/create-link.ts#L199-L208)) | scheduled job as TTL |
| A/B test completion | testCompletedAt set | QStash schedule ([ab-test-scheduler.ts](../../../apps/web/lib/api/links/ab-test-scheduler.ts)) | time-based state transition via queue |
| Request logs | every API call | `waitUntil(captureRequestLog)` ([workspace.ts#L486-L499](../../../apps/web/lib/auth/workspace.ts#L486-L499)) | best-effort |

38 cron directories under [app/(ee)/api/cron/](../../../apps/web/app/(ee)/api/cron/) consume QStash/Vercel-cron triggers, each verifying the QStash signature ([verify-qstash.ts](../../../apps/web/lib/cron/verify-qstash.ts)).

## The reliability concepts, defined by this repo's choices

- **Idempotency** (same operation twice = same result): provided by *unique keys*, not by flags — `(domain,key)` and `(projectId, externalId)` uniques make create-retries safe; QStash *consumers* must be idempotent because publishes lack a `deduplicationId` (the TODO). When you add a background job here, your first question is "what happens when it runs twice?"
- **Retries**: three flavors visible — client-side (`fetchWithRetry` to Tinybird), infra-side (QStash redelivery), and DB-transient (`withPrismaRetry`, [create-link.ts#L55](../../../apps/web/lib/api/links/create-link.ts#L55)). Each is safe only because of an idempotency argument; rehearse stating that argument per site.
- **Timeouts**: `redisGlobalWithTimeout` on the redirect path ([cache.ts#L85](../../../apps/web/lib/api/links/cache.ts#L85)) — a slow cache is worse than no cache on a latency budget. Investigate its actual timeout value in [lib/upstash](../../../apps/web/lib/upstash/) before quoting one.
- **Backpressure/load-shedding**: dedup-cache failure *drops the click* rather than hammering downstream ([record-click.ts#L103-L107](../../../apps/web/lib/tinybird/record-click.ts#L103-L107) — "if redis fails, return null so we don't overwhelm TB/MySQL"). Deliberate data loss as protection — name it as a choice.
- **Compensation/outbox**: absent. The DB write and its side effects are not linked by any durable record; a crash between them loses the side effects silently. This is the architecture's main honest gap (see [critique](06-architecture-critique.md)).
- **Failure visibility**: named-operation error logs ([record-click.ts#L233-L262](../../../apps/web/lib/tinybird/record-click.ts#L233-L262)), webhook failure callbacks + auto-disable ([lib/webhook/failure.ts](../../../apps/web/lib/webhook/failure.ts)). Weakest where effects silently skip (webhook cache miss L303-L309, usage-limit gate L280-L286).

## Side effects in risky places (flagged, not proven bugs)

1. Quota enforcement (`throwIfLinksUsageExceeded`) depends on counters maintained by best-effort async writes — sustained failure = quota bypass (slow, bounded, but real).
2. Webhook dispatch depends on a cache with no DB fallback — silent skip on eviction.
3. The optimistic image URL in the create response can 404 briefly (or forever, if the R2 upload failed) — a contract that sometimes lies.
4. Billing-adjacent counters (workspace usage) ride the same lossy pipeline as vanity counters — same mechanism, very different blast radius.

## Drills

1. For each row of the side-effect map, write "runs twice → ?" in one sentence. Which rows lack an idempotency story?
2. Design (on paper) the minimal outbox for `link.created` webhooks: table columns, writer, publisher cron, and what you do about ordering. What does it cost per create?
3. Find the QStash consumer for anonymous-link deletion (`/api/cron/links/delete`) and verify it's idempotent (what if the link is already gone?).

Interview angle: this file is the source for "design a webhook system," "what is idempotency," and "how do retries go wrong" — [08/03 Q9](../08-interview-prep/03-api-and-data-modeling-questions.md), [08/01 Q13](../08-interview-prep/01-js-ts-node-deep-dive.md), [08/04 variation 4](../08-interview-prep/04-system-design-from-this-repo.md).
