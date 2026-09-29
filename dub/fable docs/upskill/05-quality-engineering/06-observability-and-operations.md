# Observability and Operations

**Observability** = can you explain what the system did without adding code after the fact. The test per flow: *"How would I know this broke, and how fast?"*

## What exists

| Signal | Mechanism | Anchor |
| --- | --- | --- |
| Structured logs | Axiom logger; every middleware request logged + flushed via `waitUntil` | [middleware.ts#L38-L40](../../../apps/web/middleware.ts#L38-L40), [lib/axiom/server](../../../apps/web/lib/axiom/server.ts) |
| API request logs | `captureRequestLog` on success *and* error paths | [workspace.ts#L486-L521](../../../apps/web/lib/auth/workspace.ts#L486-L521) |
| Request body logging | `withAxiomBodyLog` wrapper | [workspace.ts#L81](../../../apps/web/lib/auth/workspace.ts#L81) |
| Error reporting | errors logged in the central mapper; server-action errors → Axiom | [errors.ts#L92-L97](../../../apps/web/lib/api/errors.ts#L92-L97), [safe-action.ts#L9-L23](../../../apps/web/lib/actions/safe-action.ts#L9-L23) |
| Cache diagnostics | HIT/MISS console logs per tier | [cache.ts#L76-L94](../../../apps/web/lib/api/links/cache.ts#L76-L94) |
| Fan-out failures | named-operation rejection logs | [record-click.ts#L233-L262](../../../apps/web/lib/tinybird/record-click.ts#L233-L262) |
| Webhook delivery outcomes | QStash success/failure callbacks → callback route → event log + auto-disable | [qstash.ts#L52-L63](../../../apps/web/lib/webhook/qstash.ts#L52-L63), [lib/webhook/failure.ts](../../../apps/web/lib/webhook/failure.ts), [record-webhook-event.ts](../../../apps/web/lib/tinybird/record-webhook-event.ts) |
| Ad-hoc timing | `console.time("getAnalytics")` | [analytics route L119-L131](../../../apps/web/app/api/analytics/route.ts#L119-L131) |
| Audit trail | activity/audit logs modules | [lib/api/audit-logs](../../../apps/web/lib/api/audit-logs), [lib/api/activity-log](../../../apps/web/lib/api/activity-log) |

Not found in the files read: metrics/counters (everything quantitative rides logs), distributed tracing, explicit health checks, alerting rules (likely platform-side — Vercel/Axiom dashboards; unverifiable from the repo).

## "How would I know this broke?" — per major flow

- **Redirects failing** (worst case: product down): platform 5xx dashboards + Axiom middleware logs. A *subtle* break — wrong destination after cache-shape drift — has **no signal at all** today; only customer reports. That asymmetry (hard failures visible, wrong-answer failures invisible) is the mature observation to make.
- **Click loss**: rejected-promise logs exist, but nobody is paged by a log line; there's no ingest-rate metric to alarm on a 50% drop. Detection today ≈ a customer noticing flat analytics.
- **Webhook silence**: failure callbacks + auto-disable give per-endpoint visibility; the silent-skip paths (cache miss, usage gate — [record-click.ts#L266-L309](../../../apps/web/lib/tinybird/record-click.ts#L266-L309)) emit nothing. First improvement: counters on skips ([critique change #1](../03-architecture-and-patterns/06-architecture-critique.md)).
- **Counter drift**: invisible by construction until someone diffs the stores; hence the reconciliation-cron proposal.
- **Cron failures**: QStash dashboard shows failed deliveries; the `with-cron` wrapper ([lib/cron/with-cron.ts](../../../apps/web/lib/cron/with-cron.ts) — read before citing details) centralizes job error handling.

## Deploy and rollback (inferred — platform-side)

Vercel deploys per commit with instant rollback to a previous deployment; `instrumentation.ts` runs at boot. The data caveat for rollback: schema is db-push (state, not versioned migrations in-repo), so **code rollback ≠ schema rollback** — additive-only schema changes are what make rollbacks safe. Redis cache survives deploys (external), so a rollback can pair old code with new cached shapes — the reverse of the drift problem; TTLs bound it.

## Transferable checklist for any feature you ship

- [ ] What log line proves it ran? What proves it ran *correctly*?
- [ ] What's the metric a 50% failure rate would move, and who sees it?
- [ ] Do silent-skip / early-return paths emit anything?
- [ ] Can support answer "what happened to request X" with tools they have?
- [ ] Does rollback of this change leave data (cache entries, queue messages, schema) that the old code can't handle?

Drill: pick the anonymous-link-deletion cron (`/api/cron/links/delete`). Write its "how would I know" answer: what QStash shows, what the job logs, and the failure mode where anonymous links silently pile up forever. Then design the one metric that would catch it.

Interview angle: "how do you monitor your services?" — the strong pattern is signals (logs/metrics/traces) → per-flow "how would I know" → one honest gap you've seen and fixed. This file gives you the middle part with real anchors; [08/06](../08-interview-prep/06-behavioral-star-stories.md) story 6 turns it into a STAR.
