# Senior Build Projects

Six projects, 2 days–4 weeks, each realistic enough that a maintainer *might* accept it (several map to visible TODOs). Full design discipline required: write the design doc, get it "reviewed" (rubber-duck or a peer), then build. Every plan must include: problem, product value, architecture decisions, likely files, migration plan, test plan, security plan, performance plan, rollout/rollback, open questions.

---

## Project 1: Webhook outbox — durable domain events (2–3 weeks)

Problem: DB writes and webhook publishes aren't atomically linked; a crash between them silently loses events ([critique risk #1](../03-architecture-and-patterns/06-architecture-critique.md)).
Product value: contractual "you will receive every event" for webhook customers; unlocks event replay.
Design checklist: outbox table (id, eventType, payload JSON, createdAt, publishedAt, attempts); write inside the same Prisma nested-write/transaction as the entity change; publisher cron drains → QStash with deduplicationId; ordering guarantees (per-resource FIFO or documented best-effort); retention/cleanup.
Likely files: new `packages/prisma/schema/outbox.prisma`, `lib/webhook/outbox.ts`, new cron under `app/(ee)/api/cron/webhooks/`, edits at each `sendWorkspaceWebhook` call site ([links route L87-L95](../../../apps/web/app/api/links/route.ts#L87-L95) etc.).
Migration plan: dual-write behind flag → verify parity via M3-style comparison → cut over → remove direct publishes.
Test plan: kill-the-publisher chaos test; duplicate-drain idempotency; integration test on delivery.
Security: payloads at rest now — PII retention policy for the table.
Performance: one extra insert per event; drain batch sizing.
Rollout/rollback: flag per stage; rollback = flip flag (direct-publish path stays until deleted).
Open questions: does the click-webhook path (high volume, [record-click.ts#L294-L359](../../../apps/web/lib/tinybird/record-click.ts#L294-L359)) go through the outbox or stay fire-and-forget? (Recommend: stay — document the two-tier guarantee.)
Stretch: customer-facing replay API.
Interview story potential: a complete "designed and migrated to an outbox" story — the single most reusable senior systems story in fullstack interviews.

## Project 2: Webhook replay & delivery dashboard (1–2 weeks)

Problem: failed webhook deliveries are visible internally (callbacks) but customers can't see or replay them.
Product value: self-serve debugging; fewer support tickets.
Design: deliveries listing API reading the webhook-event log ([record-webhook-event.ts](../../../apps/web/lib/tinybird/record-webhook-event.ts), [get-webhook-events.ts](../../../apps/web/lib/tinybird/get-webhook-events.ts)); replay endpoint re-publishing a stored payload (idempotency: same event id + new delivery id); UI page under the workspace settings webhooks section ([ui/webhooks](../../../apps/web/ui) — locate exact dir first).
Security plan: replay is a write — `webhooks.write` permission; signature re-signing with current secret; rate-limit replays.
Test plan: integration tests on list/replay; replay of a delivery whose webhook was since disabled (should 4xx? decide).
Rollout: read-only listing first, replay second.
Open questions: retention window of Tinybird event log bounds replayability.
Interview story potential: "I built an operational self-serve tool on an event log" — great product-engineering story.

## Project 3: Link import from a competitor (Rebrandly-style) (2–4 weeks)

Problem: migration friction; the repo already has import infra (Bitly/Short.io — [app/(ee)/api/cron/import](../../../apps/web/app/(ee)/api/cron/import), [lib/api/links/bulk-create-links.ts](../../../apps/web/lib/api/links/bulk-create-links.ts)).
Product value: acquisition funnel — imports are how customers arrive.
Design: study one existing importer end-to-end first (auth handshake, pagination, QStash-chunked continuation, error logging via [log-import-error.ts](../../../apps/web/lib/tinybird/log-import-error.ts)); mirror its shape exactly; map foreign features to Dub's (or record unmappable ones in the import error log).
Migration/perf: chunked, resumable, idempotent per chunk (re-run a chunk safely — upsert semantics by `(domain,key)`).
Test plan: fixture-driven importer unit tests (this code is *not* black-box testable — a legitimate place to introduce unit testing to the repo).
Rollout: hidden behind feature flag; dogfood with a fake account.
Open questions: rate limits of the source API; domain-verification interplay.
Interview story potential: "I built a resumable, idempotent bulk importer" — checks the async/reliability box hard.

## Project 4: Unit-test foundation for the domain layer (1–2 weeks, high acceptance odds)

Problem: the subtlest logic (processLink branching, scope intersection, cache-key normalization, fan-out labeling) has zero fast tests ([testing strategy](../05-quality-engineering/01-testing-strategy.md)).
Product value: contributor velocity and CI cost; catches the bug classes the live-API suite can't.
Design: introduce a `tests/unit/` vitest project (separate config — no `E2E_*` env requirement, parallel, sub-second); pick pure-ish seams: `processLink` (mock the few DB touches or extract them), `mapScopesToPermissions`, `combineTagIds`, `case-sensitivity`; document the seam-choice rationale.
Migration: none — additive. The politics *are* the project: propose via issue first; maintainers may prefer a different structure.
Test plan: the tests are the deliverable; target the branchiest functions (measure with `npx vitest --coverage` on the unit project only).
Rollout: CI wiring in `turbo.json` test pipeline.
Open questions: where's the line between "extract for testability" (good) and "refactor everything" (rejected PR)?
Interview story potential: "I introduced a unit-testing layer to a mature codebase, including the social engineering" — behavioral gold.

## Project 5: Self-hosting hardening — remove env-conditional security skips (1–2 weeks)

Problem: QStash signature verification and geo/IP handling short-circuit off-Vercel ([verify-qstash.ts#L18-L21](../../../apps/web/lib/cron/verify-qstash.ts#L18-L21), [record-click.ts#L118-L129](../../../apps/web/lib/tinybird/record-click.ts#L118-L129)) — fine for dub.co, holes for self-hosters.
Product value: the repo is AGPL and self-hostable; security parity matters to that audience.
Design: replace `VERCEL === "1"` checks with capability flags (`CRON_SIGNATURE_REQUIRED`, geo-provider abstraction) defaulting to strict; document a local-dev override; audit *all* `process.env.VERCEL` sites first (grep — the count is the real scope).
Security plan: default-closed; loud boot warning when relaxed.
Rollout: behind env with strict default only on new installs (changing defaults under existing self-hosters is a breaking change — semver/changelog discipline).
Open questions: how many code paths genuinely need Vercel primitives (`waitUntil`, `geolocation`) vs incidentally use them — this becomes a portability map.
Interview story potential: "I audited and hardened environment-conditional security paths" — rare, memorable.

## Project 6: Load-test rig + latency budget for the redirect path (2 days–1 week)

Problem: the hot path's performance characteristics are institutional knowledge, not artifacts.
Product value: regression protection for the product's core metric.
Design: k6/artillery scenario hitting a local/staging deployment: cold vs warm cache, password links (extra DB read), geo/device branches; output p50/p95/p99 per branch; document the budget and the current numbers in-repo.
Test plan: the rig is the test; wire a smoke variant into CI if a staging target exists.
Security: never against production; synthetic domains only.
Open questions: how faithful is local (no Vercel geo, no fluid instances) — document the deltas honestly ([verification-log discipline](../09-reference/verification-log.md)).
Stretch: flame-graph one cache-miss redirect.
Interview story potential: "I built the load-test rig and latency budget for a service doing X redirects" — concrete numbers to quote in system-design rounds.
