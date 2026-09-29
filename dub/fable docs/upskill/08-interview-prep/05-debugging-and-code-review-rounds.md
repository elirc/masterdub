# Debugging and Code Review Rounds (Timed Simulations)

Practical rounds reward *narrated method* over silent brilliance. Each simulation below has a time box, interviewer follow-ups, and a rubric. Do them aloud, ideally with a friend playing interviewer — reading them silently is worth maybe 20% of the value.

## How to narrate (the meta-skill)

State the pipeline → pick the halving observation → say what each result would mean **before** you look → look → repeat. Interviewers grade the loop, not the bug. When stuck, say what you'd measure next — never go quiet.

---

## Debugging Round 1: The stale redirect (30 min)

Setup (interviewer reads): "A customer edited their link's destination an hour ago. Clicks still go to the old URL. Here's the codebase" — you get [lib/api/links/cache.ts](../../../apps/web/lib/api/links/cache.ts), [lib/middleware/link.ts](../../../apps/web/lib/middleware/link.ts), and [lib/api/links/update-link.ts](../../../apps/web/lib/api/links/update-link.ts).
Your target performance: within 10 minutes, articulate the three cache tiers with TTLs and conclude that one-hour staleness exceeds every TTL → the write path must have failed to invalidate → hypothesize key-identity change (domain/key edit) or a failed cache set inside `waitUntil`.
Interviewer follow-ups: "The customer renamed the key during the edit. Now what exactly happened?" (old-key cache entry still live for 24h, or explicitly expired — check `update-link.ts` for old-key handling); "How would you prevent this class of bug?" (test that edits both keys; invalidate old key on identity change).
Rubric — Basic: finds the tiers. Solid: reasons from TTL bounds to "invalidation bug, not staleness." Strong: identifies key-identity as the likely culprit *before* opening update-link.ts, and proposes the regression test.

## Debugging Round 2: The missing webhook (30 min)

Setup: "Customer says `link.clicked` webhooks stopped Tuesday. `link.created` webhooks still arrive." Files: [record-click.ts#L264-L359](../../../apps/web/lib/tinybird/record-click.ts#L264-L359), [qstash.ts](../../../apps/web/lib/webhook/qstash.ts), [failure.ts](../../../apps/web/lib/webhook/failure.ts).
Target: model producer/transport/consumer; note `link.clicked` has a *different producer path* than `link.created` (click pipeline vs API route) — so the asymmetry localizes the bug to the click path's guards: usage limit exceeded, webhook auto-disabled, or webhook-cache miss (the TODO).
Follow-ups: "QStash shows zero publishes for this webhook since Tuesday. Which guard do you check first and how?" (disabledAt in DB — one query; then usage vs usageLimit; then cache membership); "The cache was flushed Tuesday during a Redis migration — connect the dots."
Rubric — Strong: uses the event-type asymmetry to skip half the search space in the first two minutes; names the silent-skip design flaw and the metric that would have caught it.

## Debugging Round 3: Analytics shows zero clicks for a busy link (30 min)

Setup: "Support escalation: link obviously getting traffic (customer sees referrals) but analytics shows ~zero." Files: [record-click.ts](../../../apps/web/lib/tinybird/record-click.ts) guards section, [link.ts](../../../apps/web/lib/middleware/link.ts) click-recording branches.
Target: enumerate recording suppressors in order: `dub-no-track` header/param, bot detection false-positives (their traffic comes from an in-app browser with a weird UA?), dedup (all clicks behind one corporate NAT + same UA → one identity hash → 1 click/hour), password gate, and the `?qr=1`/trigger classification. The NAT+dedup hypothesis fits "busy but ~zero" best.
Follow-ups: "It's a link on posters inside one university campus." (single egress IP → identityHash collisions → dedup eats nearly everything; discuss whether that's a bug or the fraud-protection working as designed — and what you'd change: per-QR trigger exemption? shorter dedup for qr triggers?)
Rubric — Strong: generates ≥4 suppressor hypotheses before choosing, picks the discriminating evidence for each (UA strings, IP distribution, trigger values in Tinybird), and ends with a product-judgment discussion, not just a fix.

## Debugging Round 4: Works locally, broken deployed (25 min)

Setup: "A contributor's cron endpoint runs fine locally but 400s in production with 'Upstash-Signature header not found'... no wait, it's the opposite: it works in prod but their local QStash-driven flow silently never fires. Explain both directions."
Target: [verify-qstash.ts#L18-L21](../../../apps/web/lib/cron/verify-qstash.ts#L18-L21) — verification skipped when `VERCEL !== "1"`; locally QStash can't reach localhost at all without the ngrok setup (`APP_DOMAIN_WITH_NGROK` — [qstash.ts#L53](../../../apps/web/lib/webhook/qstash.ts#L53)); prod requires real signatures. The general lesson: env-conditional code means environments exercise *different programs*.
Follow-ups: "How do you test signature verification at all, then?"; "What's your policy on `if (isProd)` branches?"
Rubric — Strong: names both mechanisms quickly, generalizes to the env-divergence principle, proposes capability flags with strict defaults ([project 5](../06-contribution-practice/03-senior-build-projects.md)).

---

## Review Round 1: The clickLimit PR (30 min — from [kata 1](../04-code-reading-gym/04-review-katas.md))

Simulate: interviewer plays the author, mildly defensive. You must land the blocking finding (cached shape not updated → feature inert for hot links) *while keeping them collaborative*, and correctly tier the race-window finding as important-not-blocking.
Follow-ups they'll throw: "But I tested it and it works!" (their test created the link fresh — cache primed with the new field; the bug only shows on links cached *before* deploy or edited via paths that don't re-prime — explain warmly).
Rubric — Strong: finds it, explains the hit-vs-miss asymmetry in one breath, offers the invalidation pattern by name and location.

## Review Round 2: The "delete redundant folder check" PR (20 min — from [kata 2](../04-code-reading-gym/04-review-katas.md))

The trap round: a perf win that deletes authorization. Target performance: block it in the first five minutes, cite the second authz dimension ([folder ACLs](../../../apps/web/lib/folder/permissions.ts)), point at the existing test that should have caught it ([folder-link-access.test.ts](../../../apps/web/tests/links/folder-link-access.test.ts)), then *still engage the perf problem seriously* with two alternatives (cache folder permissions; measure first). Blocking without helping is a junior reviewer's version of this round.

## Review Round 3: The bulk-delete PR (30 min — from [kata 8](../04-code-reading-gym/04-review-katas.md))

Full-spectrum review: unboundedness, per-item authz, side-effect parity, webhook loops in-request. Practice writing the *summary comment* — three sentences that tier all findings and end with a concrete path forward ("chunk via the existing cron workflow pattern under app/(ee)/api/cron/domains").
Rubric — Strong: your summary comment alone would let the author fix everything without reading the inline comments.

---

Scoring yourself across all rounds: keep a log — (1) minutes to first correct hypothesis, (2) hypotheses generated before committing, (3) did you narrate continuously, (4) did you end with a regression-prevention proposal. Improvement across attempts matters more than any single score.
