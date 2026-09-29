# Learning Rubrics

Observable behaviors, not vague traits. Use the checklists honestly; "interview-ready" means you can perform the mid-level column *aloud, under time pressure, with anchors*.

## Skill: Codebase navigation

| Level | Observable behavior | Interview-ready when… |
| --- | --- | --- |
| Junior | Finds a named file with search; explains one function's happy path | you can locate any flow's entry point in <2 min |
| Mid | Traces a feature across layers unaided; predicts file contents before opening; knows what to *ignore* | you can narrate the redirect or create flow with 5+ file anchors from memory |
| Senior | Maps ownership and boundaries; identifies which files are load-bearing vs incidental; teaches the map | you can answer "how would you ramp onto our codebase?" with a method you've actually run |

## Skill: JS/TS/async

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Uses async/await correctly on happy paths | — |
| Mid | Chooses all/allSettled/sequential deliberately; explains floating-promise hazards; types boundaries with Zod/unions | 12/14 cards in [08/01](../08-interview-prep/01-js-ts-node-deep-dive.md) at the "mid adds" register |
| Senior | Designs failure semantics (retry/idempotency/visibility) per effect; removes `any` at boundaries systematically | you can defend a fire-and-forget vs queue decision with a failure-mode table |

## Skill: Data modeling & persistence

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | CRUD with an ORM; adds columns | — |
| Mid | Designs uniques/indexes per query pattern; writes expand-migrate-contract plans; knows transaction boundaries | you can whiteboard the links/tags/workspace schema and defend every unique |
| Senior | Classifies denormalizations by repair story; plans cross-store consistency; prices migrations in lock time and rollback risk | your `clickLimit` design ([03/02 drill](../03-architecture-and-patterns/02-data-model-and-persistence.md)) covers cache-shape + backfill + rollback unprompted |

## Skill: Security & multi-tenancy

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Adds auth checks where told | — |
| Mid | Recites and applies the authz ladder; spots missing tenant scope in review; distinguishes authn/authz/isolation | you catch kata 2 and kata 5's blockers cold |
| Senior | Designs structural defenses (scoped helpers, intersect-not-union scopes); reasons about failing open vs closed; audits parallel surfaces | you can explain scope∩role and 404-vs-403 with anchors, then name where the convention can still break |

## Skill: Testing & debugging

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Writes happy-path tests; debugs by print + guess | — |
| Mid | Tests contracts (codes/shapes), cleans up state, polls async effects; debugs by halving the pipeline aloud | debugging rounds 1–2 in [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) at "Solid" |
| Senior | Chooses test layers by failure class; builds observability before fixes; ends every debug with regression prevention | you can critique this repo's live-API test strategy fairly (both directions) in 3 minutes |

## Skill: Review & collaboration

| Level | Observable | Interview-ready when… |
| --- | --- | --- |
| Junior | Approves working code; comments on style | — |
| Mid | Reviews in layers; tiers findings blocking/important/nit; writes reproducible bug reports | your kata reviews land the blocker AND keep the author collaborative |
| Senior | Blocks on evidence while co-owning the author's problem; writes RFCs with real alternatives; prices maintainers' costs | story 5 and story 8 in [08/06](../08-interview-prep/06-behavioral-star-stories.md) are true stories you can tell |

## Self-assessment checklist (run monthly)

- [ ] I re-derived a flow I hadn't read in 2+ weeks and my recall was ≥80% correct.
- [ ] I completed ≥1 ticket this month and wrote its Risks section without prompting.
- [ ] I found something in this repo the curriculum doesn't mention (the real graduation signal).
- [ ] My "what would you improve in a system you know?" answer has changed since last month (judgment growing, not calcifying).
- [ ] I can still fail productively: my last wrong hypothesis was written down, and I know what corrected it.
