# Mid-Level Feature Tickets

Ten cross-layer tickets. Rule: **write a half-page design note first** (data model delta, API delta, cache/async implications, test plan, rollback) and self-review it against the [security checklist](../05-quality-engineering/05-security-checklist.md) before writing code. Each includes Risk and Rollback sections — practice filling them like you'd have to in a real RFC.

---

## Ticket M1: Per-link click limit with auto-disable

Difficulty: Medium-Hard — 2–4 days
Story: As a workspace member, I can set `clickLimit` on a link; once reached, the link behaves as disabled.
Layers: schema ([link.prisma](../../../packages/prisma/schema/link.prisma)) + zod schemas + processLink validation + redirect path + **cached shape** (`formatRedisLink` — the trap from [kata 1](../04-code-reading-gym/04-review-katas.md)) + link builder UI + tests.
Read first: how `expiresAt` flows end-to-end (the sibling feature) — [trace 2](../04-code-reading-gym/02-trace-tables.md).
Risk: hot-path change; counter reads are eventually consistent so the limit is approximate — state the tolerance in the design note.
Rollback: column is additive + nullable; feature-flag the UI; redirect check must no-op on null.
Interview story potential: "I shipped a feature through a cached hot path and had to reason about eventual-consistency tolerances."

## Ticket M2: Webhook cache DB fallback (resolve the TODO)

Difficulty: Medium — 1–2 days
Story: As a webhook customer, my `link.clicked` events survive Redis cache evictions.
Layers: [record-click.ts#L303-L309](../../../apps/web/lib/tinybird/record-click.ts#L303-L309) + [webhook cache](../../../apps/web/lib/webhook/cache.ts) + a metric/log for fallback hits.
Design questions to answer first: on fallback, repopulate the cache? partial `mget` hits (3 of 5 ids found)? DB read cost per click — bound it.
Risk: adds DB load to the click pipeline; must not turn cache-miss storms into DB storms (consider a short negative-cache).
Rollback: pure code path addition; revert cleanly.
Interview story potential: "I closed a silent-data-loss gap flagged by a TODO, with load math to justify the approach."

## Ticket M3: Drift-reconciliation cron for link counters

Difficulty: Medium-Hard — 3–5 days
Story: As an operator, I get a nightly report of divergence between `Link.clicks` and Tinybird counts on a sample of links.
Layers: new cron under [app/(ee)/api/cron](../../../apps/web/app/(ee)/api/cron) (copy an existing job's shape + [verify-qstash](../../../apps/web/lib/cron/verify-qstash.ts)), Tinybird query via an existing pipe or a new one, Axiom log/report output.
Design questions: sample size vs cost; what threshold is "drift"; report only, or repair? (Report-only first — measurement before medicine.)
Risk: read-only → low; main risk is a misleading metric (dedup rules make raw comparisons unequal — clicks in MySQL count exactly what recordClick incremented, so define the comparable quantity carefully).
Rollback: delete the cron.
Interview story potential: "I built the measurement that turned a hypothetical consistency risk into a quantified one" — outstanding senior-signal story.

## Ticket M4: QStash deduplicationId for webhook publishes

Difficulty: Medium — 1–2 days
Story: As a webhook consumer, retried publishes don't double-deliver.
Layers: [qstash.ts#L69-L89](../../../apps/web/lib/webhook/qstash.ts#L69-L89) (the payload already carries `payload.id`), QStash API docs for the header/field, docs update for consumers.
Design questions: dedup window semantics at QStash; does dedup interact with the test-mode delay (L88)?
Risk: over-dedup (two *distinct* events sharing an id — verify id generation uniqueness in [prepareWebhookPayload](../../../apps/web/lib/webhook/transform.ts)).
Rollback: remove the field.
Interview story potential: "I added idempotent delivery semantics to a webhook system."

## Ticket M5: Skip-path metrics for the click pipeline

Difficulty: Medium — 2 days
Story: As an operator, I can alert on: bot drops, dedup-cache failures, webhook cache misses, usage-gate skips.
Layers: [record-click.ts](../../../apps/web/lib/tinybird/record-click.ts) counters (Axiom events or a Tinybird ops datasource), dashboards doc.
Design questions: cardinality (per-workspace tags?); sampling for high-frequency signals.
Risk: hot-path overhead — batch/fire-and-forget only.
Rollback: remove emission calls.
Interview story potential: "I instrumented silent failure paths before touching them — observability-first refactoring."

## Ticket M6: Link transfer between folders honoring both ACLs

Difficulty: Medium — 2–3 days
Story: As a member, moving a link between folders requires write access to *both* source and destination folders.
Layers: update-link path ([update-link.ts](../../../apps/web/lib/api/links/update-link.ts)) + [folder/permissions.ts](../../../apps/web/lib/folder/permissions.ts) + tests modeled on [folder-link-access.test.ts](../../../apps/web/tests/links/folder-link-access.test.ts).
First verify: current behavior — is source-folder access already checked on move, or only destination (`skipFolderChecks` flags in [process-link.ts#L31](../../../apps/web/lib/api/links/process-link.ts#L31) hint at subtlety)? If both are checked, this becomes a test-only ticket proving it.
Risk: authorization change — over-restricting breaks existing workflows; under-restricting is a leak.
Rollback: behind a check-flag.
Interview story potential: "I audited and hardened a two-sided permission check" — perfect for security-round behaviorals.

## Ticket M7: Friendly duplicate-key error

Difficulty: Medium — 1–2 days
Story: As an API user creating a taken `(domain, key)`, I get `conflict` with a clear message, not a generic 422.
Layers: race window analysis ([trace 4](../04-code-reading-gym/02-trace-tables.md)): keyChecks in processLink catch most, but concurrent creates reach Prisma P2002 → catch it specifically in [links route L100-L105](../../../apps/web/app/api/links/route.ts#L100-L105) or in a shared helper; error-code mapping ([error-codes.ts](../../../apps/web/lib/api/error-codes.ts)); tests for both the pre-check and the race path (the race is untestable black-box — unit-test the mapper).
Risk: public error-contract change (409 vs 422) — check the OpenAPI spec and SDKs; may need to keep 422 and improve only the message. Investigate what `ErrorCodes.conflict` maps to and whether SDKs branch on it.
Rollback: message-only fallback.
Interview story potential: "I improved an API error contract without breaking existing clients — including the compatibility analysis."

## Ticket M8: Workspace-level default `doIndex` policy

Difficulty: Medium-Hard — 3–4 days
Story: As a workspace owner, I can set whether new links default to indexable.
Layers: workspace schema/settings + processLink default resolution + settings UI + API docs. Follow how another workspace default (e.g. `defaultFolderId` on ProjectUsers — [workspace.ts#L354](../../../apps/web/lib/auth/workspace.ts#L354)) flows.
Design questions: precedence (explicit link value > workspace default > global default); does changing the workspace default retroactively affect existing links (it must not — say why in the note).
Risk: SEO implications for customers if defaults flip — additive setting, default preserving current behavior.
Rollback: setting ignored = old behavior.
Interview story potential: "I designed a three-level default-resolution scheme for a customer-facing setting."

## Ticket M9: Export links to CSV respecting active filters

Difficulty: Medium — 2–3 days
Story: As a member, the links list's current filter set can be exported.
Layers: reuse [format-links-for-export.ts](../../../apps/web/lib/api/links/format-links-for-export.ts) and the export patterns ([app/api/links/export](../../../apps/web/app/api/links/export) exists — first map what it already does; the ticket may become "add missing filters/columns"); UI trigger; pagination/streaming for MEGA workspaces.
Risk: unbounded export = memory blowup (see kata 8 lessons); folder ACLs must apply to exports exactly as to lists.
Rollback: UI-flagged.
Interview story potential: "I extended an export path with scale guards and permission parity."

## Ticket M10: Rate-limit headers on 429s everywhere

Difficulty: Medium — 1–2 days
Story: As an API consumer, every 429 includes `Retry-After` and the standard rate-limit headers, on both token and session paths.
Layers: [rate-limit-request.ts](../../../apps/web/lib/auth/rate-limit-request.ts) + the two call sites in [workspace.ts#L248-L265](../../../apps/web/lib/auth/workspace.ts#L248-L265), [#L324-L339](../../../apps/web/lib/auth/workspace.ts#L324-L339) — verify current header behavior first (headers are set before the throw; do they survive onto the error response? follow `responseHeaders` into [handleAndReturnErrorResponse](../../../apps/web/lib/api/errors.ts)).
Risk: header contract clients may parse — additive only.
Rollback: trivial.
Interview story potential: "I traced header propagation through an error path and fixed a DX gap" — small but crisply narratable.
