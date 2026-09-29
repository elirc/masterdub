# Two-Week Cram Plan

For the candidate with an interview in ~14 days. Assumes 2–3 focused hours on weekdays, 4–5 on weekends. Everything references files in this curriculum; nothing requires a running instance. **Speak answers aloud** — silent review is roughly a third as effective for interview performance.

## Week 1 — build the evidence base

**Day 1 (map):** [00-fast-track.md](../00-fast-track.md) Saturday-morning sections + first 10 files. Evening: teach-back — 3 minutes aloud on "what happens when someone clicks a Dub link," then compress to 90 seconds.
**Day 2 (flows):** [key flows](../01-codebase-cartography/05-key-flows.md) Flows 1, 2, 5 with the files open. Do Flow 1's drill. Aloud: the guard order and cache tiers, from memory.
**Day 3 (JS/TS cards):** [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) Q1–Q7, 90 seconds each aloud *before* reading the card. Log which were weak.
**Day 4 (JS/TS + async):** Q8–Q14, plus reread [03/04-side-effects](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md). Do its drill 1 (the runs-twice table).
**Day 5 (frontend cards):** [02-frontend-framework-questions.md](02-frontend-framework-questions.md) all 10, with [link-card.tsx](../../../apps/web/ui/links/link-card.tsx) open for Q2–Q4.
**Day 6 (system design, first full run):** [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) — whiteboard "design a link shortener" for 40 minutes *without* the file, then grade against it. Gaps → flashcards.
**Day 7 — checkpoint (self-assessment):**
- [ ] 90-second redirect narration: fluent, includes cache tiers + 302 reasoning?
- [ ] Can recite the authz ladder (shape→identity→abuse→tenant→permission→plan→business→resource→folder)?
- [ ] 10/14 JS cards at mid-level or better?
- [ ] System-design run covered schema, cache, click pipeline, tenancy without prompting?
Below the bar → repeat the weakest day before proceeding; drop Day 10's second half to make room.

## Week 2 — pressure and polish

**Day 8 (API/data cards):** [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) all 12. Do Q3's whiteboard drill for real (5 min, then diff against [link.prisma](../../../packages/prisma/schema/link.prisma)).
**Day 9 (debugging round 1+2, timed):** [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) rounds 1 and 2, full 30 minutes each, narrating aloud. Log time-to-first-hypothesis.
**Day 10 (review rounds):** review rounds 1 and 2, writing the actual review text. Then [07/01-code-review-mindset](../07-career-and-collaboration/01-code-review-mindset.md) — memorize the five layers.
**Day 11 (stories):** [06-behavioral-star-stories.md](06-behavioral-star-stories.md) — pick your 5 (must include a conflict and a mistake story), fill every blank with *your* specifics, rehearse each twice under 2 minutes. Record yourself once; listen (painful, works).
**Day 12 (system design, second run + variations):** full run again — should be visibly tighter — then variations 2 and 4 (10x traffic; webhook endpoint down) for 5 minutes each aloud.
**Day 13 (mock day):** simulate the loop: 40-min system design + 30-min debugging round 3 + 30-min behavioral (5 stories, shuffled prompts). Ideally with a human; else record.
**Day 14 — checkpoint + taper:**
- [ ] All 5 stories under 2 min, ending with impact?
- [ ] Debugging narration continuous (no silent stretches >30s)?
- [ ] Can you answer "tell me about a system you know well" for 10 minutes with anchors? (This is your superpower — Dub *is* the answer.)
- [ ] One honest "what would you improve in that system?" ready ([critique](../03-architecture-and-patterns/06-architecture-critique.md) — pick two: silent-skip metrics, webhook dedup)?
Taper: evening off. Interview-day recall sheet: the cache tiers + TTLs, the authz ladder, 302-vs-301, the fan-out list, scope∩role, and your 5 story titles.

## If you only have one week

Days 1, 2, 6, then 9, 11, 13, 14. The cards you skip, skim only the "Mid-level answer adds" lines — they're the target register.
