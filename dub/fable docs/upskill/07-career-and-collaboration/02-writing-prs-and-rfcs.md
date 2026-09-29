# Writing PRs and RFCs

## The PR description contract

Five sections, always, even for small diffs:

```markdown
## What
One paragraph: the change, in behavior terms ("expired links with no expiredUrl now
return 404 instead of rendering the expired page for bots").

## Why
The problem/ticket, with evidence (log, issue link, anchor to the code that motivated it).

## How tested
Exact commands + what you observed. "pnpm test tests/links/expired-link.test.ts — passes;
manually verified 302 → expiredUrl via curl -I". Honesty over polish: if you couldn't
test a branch (e.g., the Redis-failure fallback), SAY SO — reviewers can then focus there.

## Risks
Blast radius (hot path? public contract? cached shape?), failure mode if you're wrong,
and the rollback (revert-safe? flag?).

## Follow-ups
What you deliberately didn't do, so scope-creep debates end in a linked ticket.
```

The "Risks" section is what separates mid-level PRs from junior PRs. For this repo the recurring risk vocabulary is: hot-path (redirect), cached-shape ([formatRedisLink](../../../apps/web/lib/middleware/link.ts#L110)), public contract (OpenAPI/webhooks/published packages), side-effect parity (create vs bulk vs update paths), and env-conditional behavior (Vercel vs local).

## Commit messages

Imperative summary ≤72 chars, body explains *why* when non-obvious. Squash noisy WIP before review. In a monorepo, prefix scope helps triage: `web/links: guard clickLimit on cached redirect path`. A reader of `git log --oneline` six months out should be able to find "the commit that changed expiry behavior."

## When to RFC (vs just PR)

RFC when any is true: public contract changes; new table or store; new background mechanism (queue/cron); security posture changes; the diff would exceed ~400 lines without prior agreement; or you'd be establishing a *pattern* others will copy. Otherwise: issue → PR.

## RFC template (tailored to this repo)

```markdown
# RFC: <title>
Status: draft | discussing | accepted | rejected
## Problem
What breaks or costs today, with anchors (file:line) and, ideally, a number.
## Proposal
The design. Include: data model delta (.prisma), API delta (zod schema diff),
cache/queue implications, and the failure-mode table (effect → runs twice? → fails silently?).
## Alternatives considered
At least two, each with the reason it loses — including "do nothing" with its real cost.
## Migration & rollout
Expand-migrate-contract steps; flags; what rollback leaves behind (cache entries,
queue messages, schema).
## Blast radius
Hot path touched? Public contract? Self-hosters? Published packages semver?
## Open questions
Genuinely open — decisions you want the reviewer to make.
```

Worked example to practice with: kata 8 (API versioning RFC) in [06/04](../06-contribution-practice/04-refactor-and-design-katas.md); the outbox design (project 1) is the full-dress rehearsal.

Interview angle: many mid-level loops include "write a short design doc" or judge you on PR hygiene in the practical round. The Risks-section habit reads instantly as experience. Behavioral: "how do you get buy-in for technical changes?" → describe issue-first + RFC-with-alternatives + measured rollout, with one of your katas as the example.
