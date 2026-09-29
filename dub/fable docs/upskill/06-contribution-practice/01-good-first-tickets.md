# Good First Tickets

Sixteen junior-sized tickets, spread across subsystems. These are **training tickets**: realistic contributions you could genuinely propose upstream (as issues first — see [07-career-and-collaboration/03-maintainer-communication.md](../07-career-and-collaboration/03-maintainer-communication.md)), sized so the diff is small and the learning is large. Do them on a branch even if you never open a PR. Anchors verified 2026-07-09; re-verify before coding.

Reminder of the bar for *any* contribution: why is it useful, why is the blast radius small, what existing pattern does it follow, how is it tested, and what would make a maintainer reject it.

---

## Ticket 1: Fix comment typo "desintation" in process-link

Difficulty: Easy — Estimated time: 15 min
Skills practiced: PR mechanics, minimal-diff discipline.
Story: As a maintainer, I want comments to be typo-free so search works ("destination" currently misses this line).
Why this is a good contribution: zero runtime risk; classic first-PR icebreaker.
Acceptance criteria:
- [ ] Comment at [process-link.ts#L105](../../../apps/web/lib/api/links/process-link.ts#L105) reads "destination".
- [ ] `grep -ri desintation apps/` returns nothing else (sweep the repo, fix all in one PR).
Read these anchors first: [process-link.ts#L105](../../../apps/web/lib/api/links/process-link.ts#L105).
Files likely touched: `apps/web/lib/api/links/process-link.ts`.
Implementation plan: 1. grep repo-wide. 2. Fix. 3. `pnpm prettier-check`.
What could go wrong: accidentally reformatting the file (keep the diff to the typo).
Suggested checks: diff shows only comment lines.
Review questions: did you sweep for other occurrences?
Interview story potential: weak alone — bundle as "how I broke the ice in a 500k-line codebase."

---

## Ticket 2: Name the fan-out operations structurally in `recordClick`

Difficulty: Medium — Estimated time: 2–3 h
Skills practiced: async patterns, refactoring under no-test-coverage, blast-radius reasoning.
Story: As an on-call engineer, I want failure logs from the click fan-out to stay correct when someone reorders the promise array.
Why this is a good contribution: the current index-based mapping ([record-click.ts#L233-L252](../../../apps/web/lib/tinybird/record-click.ts#L233-L252)) mislabels errors if the `Promise.allSettled` array ([#L179-L230](../../../apps/web/lib/tinybird/record-click.ts#L179-L230)) changes — the array already contains conditional entries (`workspaceId && ...`), which evaluate to `false` (not a promise) and can shift positions. **Investigate first**: confirm how `allSettled` treats non-promise falsy entries (they resolve as fulfilled values) and whether labels are already skewed for links without workspaceId.
Acceptance criteria:
- [ ] Each operation is declared as `{ name, run: () => Promise }` (or similar) so name and promise cannot desync.
- [ ] Log output shape unchanged (grep dashboards may depend on `"[Record click] - Rejected promises:"`).
- [ ] No change to which operations run or their concurrency.
Read these anchors first: [record-click.ts#L177-L262](../../../apps/web/lib/tinybird/record-click.ts#L177-L262).
Files likely touched: `apps/web/lib/tinybird/record-click.ts`.
Implementation plan: 1. Build `ops` array of named thunks, filtering out inapplicable ones *before* settling. 2. `Promise.allSettled(ops.map(o => o.run()))`. 3. Map rejections via `ops[i].name`.
Illustrative fake-code shape (labeled):
```ts
// Illustrative fake code: not from this repo
const ops = [
  { name: "tinybird-ingest", run: () => ingest(clickData) },
  ...(workspaceId ? [{ name: "usage-event", run: () => publishUsage() }] : []),
];
const results = await Promise.allSettled(ops.map((o) => o.run()));
```
What could go wrong: changing evaluation order from eager promises to lazy thunks alters *when* work starts — keep them started together.
Suggested checks: temporary local harness that stubs each op to reject and asserts the logged name.
Review questions: does the falsy-entry behavior change counts in any metrics derived from these logs?
Interview story potential: "I found an error-attribution bug in a hot-path fan-out and fixed it without behavior change" — a genuinely strong debugging/quality story.

---

## Ticket 3: Add the missing `.catch` fallback for links-usage event on create

Difficulty: Medium — Estimated time: 2 h (mostly investigation)
Skills practiced: reliability patterns, consistency auditing.
Story: As a billing owner, I want links-usage counting to survive a Redis stream outage the same way clicks-usage does.
Why this is a good contribution: [record-click.ts#L202-L214](../../../apps/web/lib/tinybird/record-click.ts#L202-L214) publishes the clicks-usage event **with a direct-SQL fallback**; the analogous links-usage publish at [create-link.ts#L210-L216](../../../apps/web/lib/api/links/create-link.ts#L210-L216) has none. Possible risk, not confirmed bug: first confirm the stream consumer and whether link usage is reconciled elsewhere (check `app/(ee)/api/cron/streams/` and `usage` cron jobs) — if it is, the right PR is a comment documenting why, not code.
Acceptance criteria:
- [ ] Either a fallback matching the established pattern, or a comment explaining the asymmetry (whichever investigation supports).
- [ ] Fallback (if added) increments the same fields the stream consumer would.
Read these anchors first: both publish sites above; `apps/web/lib/upstash/redis-streams/`.
Files likely touched: `apps/web/lib/api/links/create-link.ts`.
Implementation plan: 1. Read the stream consumer. 2. Decide fallback vs document. 3. Match the `.catch(() => conn.execute(...))` shape exactly.
What could go wrong: double-counting if the publish succeeded but its promise rejected late — mirror whatever tolerance the clicks path accepted.
Suggested checks: grep for other `publish*Event(` call sites and audit all of them; report findings in the PR.
Review questions: is at-most-once or at-least-once the right failure mode for a billing counter?
Interview story potential: "I audited a graceful-degradation pattern for consistency and found an asymmetric gap" — excellent reliability story.

---

## Ticket 4: Replace `console.time("getAnalytics")` with structured log timing

Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: observability hygiene.
Story: As an operator, I want analytics latency queryable in Axiom, not interleaved in stdout.
Why this is a good contribution: [analytics route L119-L131](../../../apps/web/app/api/analytics/route.ts#L119-L131) uses `console.time`, which is unlabelled per-request (concurrent requests share the label — Node warns and timings interleave) while the repo already has a structured logger ([lib/axiom/server](../../../apps/web/lib/axiom/server.ts)).
Acceptance criteria:
- [ ] `Date.now()` delta logged via the existing `logger` with workspaceId + groupBy dimensions.
- [ ] No `console.time` remaining in the route.
Read these anchors first: the route; how `logger` is used in [middleware.ts#L38-L40](../../../apps/web/middleware.ts#L38-L40).
Files likely touched: `apps/web/app/api/analytics/route.ts`.
What could go wrong: logging PII or high-cardinality fields; log volume cost — keep dimensions few.
Review questions: should this be sampled?
Interview story potential: one line in an observability story.

---

## Ticket 5: Document the anonymous-link-creation carve-out

Difficulty: Easy — Estimated time: 1 h
Skills practiced: reading auth code precisely; writing security-relevant comments.
Story: As a security reviewer, I want the auth wrapper's biggest exception clearly documented at the point of the check.
Why this is a good contribution: [workspace.ts#L134-L146](../../../apps/web/lib/auth/workspace.ts#L134-L146) lets requests with a `dub-anonymous-link-creation` header skip *all* auth for `POST /links`, with the real protections living elsewhere (10/day IP rate limit at [links route L58-L69](../../../apps/web/app/api/links/route.ts#L58-L69), 30-min self-delete at [create-link.ts#L199-L208](../../../apps/web/lib/api/links/create-link.ts#L199-L208)). That cross-file contract is exactly what a comment is for.
Acceptance criteria:
- [ ] Comment at the carve-out naming the two compensating controls with paths.
- [ ] Note in the comment that `@ts-expect-error` there ([L140](../../../apps/web/lib/auth/workspace.ts#L140)) exists because the handler contract is violated (no session/workspace).
Files likely touched: `apps/web/lib/auth/workspace.ts`.
Review questions: is a comment enough, or should the carve-out move into the route where its compensations live? (Write your answer in the PR description — that's the senior part.)
Interview story potential: feeds the "how do you review auth code" behavioral answer.

---

## Ticket 6: Add an integration test for expired links with `expiredUrl`

Difficulty: Medium — Estimated time: 3–4 h (needs E2E env)
Skills practiced: this repo's integration-test harness, redirect semantics.
Story: As a maintainer, I want the expiry branch ([link.ts#L234-L252](../../../apps/web/lib/middleware/link.ts#L234-L252)) covered: expired+`expiredUrl` → 302 to it; expired without → rewrite to `/expired` page.
Why this is a good contribution: the redirect decision tree has thin direct coverage (see [05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md)); expiry is deterministic and cheap to test.
Acceptance criteria:
- [ ] Test creates a link with `expiresAt` in the past via API, requests the short link with `redirect: "manual"`, asserts status/location.
- [ ] Cleanup via `h.deleteLink` in `onTestFinished` (pattern: [create-link.test.ts#L50-L56](../../../apps/web/tests/links/create-link.test.ts#L50-L56)).
Read these anchors first: [tests/utils/integration.ts](../../../apps/web/tests/utils/integration.ts), [tests/utils/http.ts](../../../apps/web/tests/utils/http.ts).
Files likely touched: new file `apps/web/tests/links/expired-link.test.ts`.
What could go wrong: link cache — the link was just created; the redirect may serve from Redis; assert after cache set (creation primes it write-through, so this should be immediate — verifying that *is* the test's value).
Review questions: does the test tolerate the `_root` 301 special case? (Not applicable — use a non-root key; say why.)
Interview story potential: "I added regression coverage to an untested hot-path branch in a real OSS redirect service."

---

## Ticket 7: Extract the repeated `recordClick`-args object in `LinkMiddleware`

Difficulty: Medium — Estimated time: 2–3 h
Skills practiced: refactoring for drift-resistance; knowing when *not* to abstract.
Story: As a maintainer, I want the six near-identical `recordClick({...})` call sites ([link.ts#L283-L296](../../../apps/web/lib/middleware/link.ts#L283-L296), [#L342-L356](../../../apps/web/lib/middleware/link.ts#L342-L356), [#L378-L392](../../../apps/web/lib/middleware/link.ts#L378-L392), [#L416-L430](../../../apps/web/lib/middleware/link.ts#L416-L430), [#L485-L499](../../../apps/web/lib/middleware/link.ts#L485-L499), [#L519-L533](../../../apps/web/lib/middleware/link.ts#L519-L533), [#L553-L567](../../../apps/web/lib/middleware/link.ts#L553-L567)) to share one local helper so a new field can't be added to five of six.
Why this is a good contribution: only `url`/`finalUrl` differs between call sites; drift here silently corrupts analytics.
Acceptance criteria:
- [ ] One local `track(finalUrl)`-style closure; call sites become one line.
- [ ] Zero behavior change (same fields, same `ev.waitUntil` timing).
Files likely touched: `apps/web/lib/middleware/link.ts` only.
What could go wrong: this file is the hottest path in the product — a maintainer may reject *any* churn here without strong motivation. Present it as risk-reduction, keep the diff mechanical, and accept "no" gracefully.
Review questions: does the closure capture anything mutable that changes between branches?
Interview story potential: "refactoring the hottest path in a redirect service — and how I argued the risk math."

---

## Ticket 8: Constant-ize the magic TTLs in click tracking

Difficulty: Easy — Estimated time: 1 h
Skills practiced: naming as documentation.
Story: As a reader, I want `60 * 5` at [record-click.ts#L174](../../../apps/web/lib/tinybird/record-click.ts#L174) to say *why* it's 5 minutes ("Tinybird ingestion lag window").
Acceptance criteria:
- [ ] Named constants (e.g. `CLICK_ID_CACHE_TTL_S`) colocated with a one-line reason; same values.
- [ ] Check for the matching TTL assumption on the read side (`/track/lead` path) and reference it.
Files likely touched: `apps/web/lib/tinybird/record-click.ts`, possibly a constants file.
Review questions: are there other consumers assuming 5 minutes that should import the constant?
Interview story potential: minor; feeds "code as communication" answers.

---

## Ticket 9: Improve the misconfigured-Authorization error with the offending prefix

Difficulty: Easy — Estimated time: 1 h
Skills practiced: DX empathy, careful non-leaky error messages.
Story: As an API user who sent `Token abc123`, I want the 400 to tell me what I sent wrong without echoing my secret.
Why this is a good contribution: [workspace.ts#L106-L113](../../../apps/web/lib/auth/workspace.ts#L106-L113) already explains the fix; adding the *scheme* the caller used (never the credential) makes it self-serve.
Acceptance criteria:
- [ ] Message includes the first token of the header (e.g. `Token`) only — never the rest.
- [ ] Existing tests still pass (`tests/` greps for the current message? verify first).
Files likely touched: `apps/web/lib/auth/workspace.ts`.
What could go wrong: leaking credentials into logs/messages — the whole point is doing this safely.
Interview story potential: feeds security-mindset answers ("even error messages are an exfiltration surface").

---

## Ticket 10: Add a `dub-no-track` test

Difficulty: Medium — Estimated time: 2–3 h
Skills practiced: negative-path testing of analytics.
Story: As a privacy owner, I want proof that `?dub-no-track=1` ([record-click.ts#L67-L70](../../../apps/web/lib/tinybird/record-click.ts#L67-L70)) suppresses click recording.
Acceptance criteria:
- [ ] Test clicks a link with and without the param, then asserts the click count via `GET /analytics` (allow for ingestion lag: poll with timeout).
Read these anchors first: how existing analytics tests poll (look in `apps/web/tests/analytics/`).
What could go wrong: the 1-hour dedup ([#L88-L108](../../../apps/web/lib/tinybird/record-click.ts#L88-L108)) makes the *second* click from the same identity invisible anyway — the test must vary identity or order the assertions to not be fooled by dedup. Realizing that is the exercise.
Interview story potential: "I wrote a test that had to out-think click deduplication" — great for testing-round interviews.

---

## Ticket 11: Document per-domain case sensitivity in the glossary/comments

Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: tracing one concept across write & read paths.
Story: As a new contributor, I want one comment block explaining that keys are lowercased at redirect time unless the domain is case-sensitive ([link.ts#L46-L52](../../../apps/web/lib/middleware/link.ts#L46-L52)) and correspondingly encoded at write time ([create-link.ts#L51-L54](../../../apps/web/lib/api/links/create-link.ts#L51-L54)).
Acceptance criteria:
- [ ] `case-sensitivity.ts` gets a file-header comment stating the invariant: *write-side encoding and read-side normalization must agree, or links 404*.
Files likely touched: `apps/web/lib/api/links/case-sensitivity.ts`.
Interview story potential: unicode/normalization edge-case story (pairs with interview Q12).

---

## Ticket 12: Type the webhook click payload instead of `@ts-ignore`

Difficulty: Medium — Estimated time: 2–4 h
Skills practiced: removing suppressions properly.
Story: As a maintainer, I want the `@ts-ignore – bot & qr should be boolean` at [record-click.ts#L353](../../../apps/web/lib/tinybird/record-click.ts#L353) resolved by fixing the type at the source, not the call site.
Acceptance criteria:
- [ ] Identify why `bot`/`qr` mismatch (clickData builds them as booleans at [#L164-L165](../../../apps/web/lib/tinybird/record-click.ts#L164-L165); the schema presumably expects numbers for Tinybird — investigate `record-click-zod.ts` / `transformClickEventData`).
- [ ] Suppression removed; `npx tsc --noEmit` clean.
Files likely touched: `lib/webhook/transform.ts` or the zod schema — follow the types.
What could go wrong: the Tinybird wire format may genuinely need 0/1 — then the fix is an explicit transform, not a type change.
Interview story potential: "how I removed a ts-ignore by finding the real type boundary" — interviewers love this.

---

## Ticket 13: Add JSDoc to `formatRedisLink` describing the cached-shape contract

Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: documenting implicit contracts.
Story: As a contributor adding a field to the redirect path, I need to know that any field read from `cachedLink` in [link.ts#L134-L151](../../../apps/web/lib/middleware/link.ts#L134-L151) must be included by `formatRedisLink` (in `lib/upstash`) — otherwise cache hits behave differently from cache misses.
Acceptance criteria:
- [ ] JSDoc on `formatRedisLink` stating the invariant and pointing at the destructure site.
- [ ] The two `as any` casts at [link.ts#L110](../../../apps/web/lib/middleware/link.ts#L110) and [#L115](../../../apps/web/lib/middleware/link.ts#L115) noted as the enforcement gap.
Interview story potential: "cache-shape drift" is a sophisticated failure mode to narrate in system-design rounds.

---

## Ticket 14: Verify and document the E2E test prerequisites in a tests/README

Difficulty: Easy — Estimated time: 2 h
Skills practiced: developer onboarding, env handling.
Story: As a new contributor, `pnpm test` fails cryptically without `E2E_BASE_URL`/`E2E_TOKEN` ([tests/utils/env.ts](../../../apps/web/tests/utils/env.ts)); a `tests/README.md` should say what's needed and that tests run against a **live deployment** with `--bail=1` and no file parallelism ([apps/web/package.json](../../../apps/web/package.json) `test` script).
Acceptance criteria:
- [ ] README documents required env, what the harness assumes (seeded `acme` workspace — [integration.ts#L43-L48](../../../apps/web/tests/utils/integration.ts#L43-L48)), and safety warnings (don't point at prod!).
Interview story potential: onboarding-empathy story for behavioral rounds.

---

## Ticket 15: Add missing operation to the fan-out failure labels

Difficulty: Easy — Estimated time: 30 min (do with/after Ticket 2)
Skills practiced: attention to detail.
Story: The `operations` label array at [record-click.ts#L237-L244](../../../apps/web/lib/tinybird/record-click.ts#L237-L244) lists 5 entries; verify it matches the settled array's *actual* length and order for every conditional combination (partner links add a 5th entry; non-workspace links produce `false` placeholders). Document or fix.
Acceptance criteria:
- [ ] A truth table in the PR description: (workspaceId? × partner?) → array shape → label correctness.
Interview story potential: merged into Ticket 2's story.

---

## Ticket 16: Propose a `typecheck` script for apps/web

Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: build tooling, monorepo scripts.
Story: As a contributor, I want `pnpm typecheck` (i.e. `tsc --noEmit`) available and wired into `turbo.json`, since `lint` (`next lint`) doesn't typecheck the whole graph and full `next build` is heavy.
Why this is a good contribution: cheap CI guard; follows existing turbo pipeline patterns ([turbo.json](../../../turbo.json)).
Acceptance criteria:
- [ ] `apps/web/package.json` gets `"typecheck": "tsc --noEmit"`; root turbo pipeline entry added.
- [ ] Document expected runtime; if the codebase currently fails `tsc --noEmit` (possible — verify!), scope the PR down to reporting the count and proposing incremental adoption.
What could go wrong: monorepos often have intentionally-excluded type errors; a failing script nobody can run green is worse than none.
Interview story potential: "I added a typecheck gate to a large monorepo and negotiated the rollout" — solid tooling story.
