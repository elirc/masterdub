# Maintainer Communication

Maintainers owe you nothing; their attention is the scarcest resource in OSS. Every template below is optimized for *their* cost of responding.

## Principles

1. **Do the thinking, then ask for a decision, not a tutorial.** "I traced X to line Y; I see two fixes, A (small, partial) and B (bigger, complete); leaning A — objections?" is answerable in one minute. "How does caching work here?" is not.
2. **Issues before PRs** for anything non-trivial. An unagreed 400-line PR is a rejection with extra steps.
3. **Minimal reproductions** or it didn't happen. For this repo: exact request (curl), expected vs actual, and whether it reproduces on a fresh clone with the documented setup.
4. **Respect the roadmap.** A feature *you* want is a proposal, not a defect. Frame value from their users' perspective.
5. **Take "no" as data.** A rejected PR that taught you why is a successful interaction.

## Template: asking a question (after being stuck properly)

```markdown
**Context:** implementing <thing>, following the pattern at <file:line>.
**What I've tried:** <2–3 bullets with results — show the dead ends>
**Specific question:** <one question, answerable without opening an IDE>
**My current guess:** <so they can just say "yes" or "no, because X">
```

## Template: proposing a feature

```markdown
**Problem:** <who hurts, how often — user language, not solution language>
**Evidence:** <issue links, forum posts, your own use case stated honestly as n=1>
**Proposal sketch:** <2–3 sentences + which existing pattern it follows, e.g.
"same shape as the Bitly importer under app/(ee)/api/cron/import">
**Scope I'd commit to:** <what you will actually build, incl. tests + docs>
**Questions for you:** <the decisions that are theirs: naming, placement, plan-gating>
```

## Template: reporting a bug

```markdown
**Summary:** <one line, behavior not blame — "expired links serve stale destination
from cache" not "your cache is broken">
**Repro:** <numbered, minimal, from clean state; exact curl/UI steps>
**Expected / Actual:** <both, one line each>
**Where I think it lives:** <file:line anchors + your hypothesis, labeled as hypothesis>
**Environment:** <self-hosted? version/commit? relevant env flags — e.g. VERCEL unset,
which changes signature verification per lib/cron/verify-qstash.ts>
**Happy to PR if you confirm the direction.**
```

## Template: responding to review pushback

```markdown
Fair — I hadn't weighed <their concern>. Two options from here:
1. <their direction, with what I'd need to change>
2. <your direction, restated with the strongest single piece of evidence>
If you still prefer 1 I'll go with it; you know the maintenance cost here better than I do.
```

That last line isn't submission — it's correctly pricing whose costs dominate. Reserve real pushback for correctness/security, where you should be politely immovable with evidence.

## Triage etiquette (when you start answering others' issues — do this, it builds reputation fast)

Reproduce before commenting; link duplicates instead of re-litigating; label uncertainty ("I *think* this is the cache tier at cache.ts#L66 — not a maintainer, but a repro is attached"). Never speak *for* the project's roadmap.

Drill: write the full feature proposal for [ticket M2](../06-contribution-practice/02-mid-level-feature-tickets.md) (webhook cache DB fallback) using the template — it has a real TODO comment as evidence ([record-click.ts#L305-L307](../../../apps/web/lib/tinybird/record-click.ts#L305-L307)), which makes it the perfect first upstream interaction.

Interview angle: "tell me about working with people you don't control" / "influence without authority" — OSS maintainer interactions are the cleanest possible STAR material, and rarer than you'd think among candidates ([08/06 story 8](../08-interview-prep/06-behavioral-star-stories.md)).
