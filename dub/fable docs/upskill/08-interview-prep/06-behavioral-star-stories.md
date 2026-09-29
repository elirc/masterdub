# Behavioral STAR Story Worksheets

Eight worksheets. Six are sourced from doing the tickets/projects in [06-contribution-practice](../06-contribution-practice/README.md) — **complete the work first, then the story is true**. Two (7, 8) work from study alone. For each: fill in your own Action/Result specifics; the skeleton gives Situation/Task and the senior-signal details interviewers listen for.

Rehearsal check applies to all: under 2 minutes? concrete numbers/files? ends with impact + what you'd do differently?

---

## Story 1: Learning a huge codebase fast

Prompts this answers: "Tell me about ramping up on unfamiliar code" / "How do you approach a legacy system?"
Source: this curriculum's Phase-1 method itself.
Situation: 500k-line open-source monorepo (Dub — link management, billions of redirects), no onboarding buddy.
Task: become productive enough to ship changes safely within weeks.
Action (yours to fill; skeleton): mapped runtime surfaces by hostname dispatch → traced two end-to-end flows (redirect, link creation) with file:line notes → built a glossary of domain-noun traps (workspace=Project in the schema) → verified understanding by predicting behavior before reading (pause-and-predict), keeping a verification log of confirmed vs assumed.
Result: e.g., "produced a system map + found two real risks (silent webhook skip, counter drift) that I turned into upstream issue reports."
Evidence to cite: your notes/PRs; the TODO at [record-click.ts#L305-L307](../../../apps/web/lib/tinybird/record-click.ts#L305-L307) you can pull up live.
Senior-signal details: method over heroics; distinguishing verified from assumed; deliverable was a map others could use.
Resume bullet: *Reverse-engineered a production link-attribution platform (Next.js/MySQL/Redis/Tinybird); documented core flows and surfaced 2 reliability risks upstream.*

## Story 2: Fixing a silent failure mode (Ticket 2 / M5)

Prompts: "A time you improved reliability" / "A bug that wasn't a bug yet."
Situation: click-analytics fan-out logged failures via an index-mapped label array; conditional entries could shift indexes, mislabeling failures exactly when debugging needed them.
Task: make failure attribution trustworthy without changing hot-path behavior.
Action: named-operation refactor; truth-table of conditional combinations; verified log-shape compatibility (dashboards grep these strings).
Result: fill in (e.g., "error logs now attribute correctly across all 4 guard combinations; zero behavior diff, verified by …").
Senior-signal details: you treated *observability output* as a contract; you considered downstream consumers of log strings.
Resume bullet: *Hardened failure attribution in a high-volume analytics pipeline (5 concurrent effects/click) with a zero-behavior-change refactor.*

## Story 3: Shipping through a cached hot path (Ticket M1)

Prompts: "Your most technically challenging feature" / "A time you had to consider systems beyond your code."
Situation: adding click limits to links, where reads come from a 3-tier cache and counters are eventually consistent.
Task: feature must work for *cached* links and tolerate approximate counts.
Action: extended the cached shape + invalidation; documented the consistency tolerance ("limit enforced within N clicks") as an explicit product decision, not an accident.
Result: fill in.
Senior-signal details: you found the cache-shape contract *before* it found you; you turned an implicit tolerance into a stated one.
Resume bullet: *Designed and shipped click-limiting through a 3-tier cache (LRU/Redis/MySQL), including invalidation and eventual-consistency semantics.*

## Story 4: Measurement before medicine (Ticket M3)

Prompts: "A time you disagreed about priorities" / "Data-driven decision."
Situation: suspected drift between denormalized counters and the event store; no one knew if it was real.
Task: quantify before proposing an expensive fix (outbox).
Action: built a sampling reconciliation cron; defined "comparable quantity" carefully (dedup rules make naive comparison wrong).
Result: fill in — either "drift negligible, saved weeks of unneeded work" or "drift real at X%, justified the fix" — **both outcomes are wins; say that**.
Senior-signal details: you designed the measurement to be cheap and the decision to be reversible.
Resume bullet: *Built drift-detection tooling for dual-write consistency between MySQL counters and a columnar event store.*

## Story 5: Pushing back in review (Review katas, practiced live)

Prompts: "Disagreement with a colleague" / "A time you blocked a change."
Situation: PR removed a "redundant" folder-permission check for performance.
Task: block a security regression without torching the relationship or dismissing the real perf problem.
Action: demonstrated the leak concretely (which links become visible to whom), cited the existing test, then co-designed the alternative (cache the permission lookup).
Result: fill in.
Senior-signal details: blocked on evidence, not authority; stayed on the author's actual problem (latency) after blocking.
Resume bullet: *Prevented an authorization regression in review while co-designing the performance fix that shipped instead.*

## Story 6: Making failures visible (Ticket M5)

Prompts: "A time you improved a process/system proactively."
Situation: several "working" paths silently skipped work (webhook cache miss, quota gates); no metrics, only optional log lines.
Task: make silence measurable before anyone touched the code.
Action: added counters on every skip path; wired the first alert; only then proposed behavior changes.
Result: fill in (ideally: "the metrics immediately showed X skips/day nobody knew about").
Senior-signal details: observability-first sequencing; you can articulate *why* ("you can't review a fix for a problem you can't see").
Resume bullet: *Instrumented silent failure paths in a click pipeline processing millions of events, enabling the first alerting on data loss.*

## Story 7: A mistake and what changed (works from study — make it yours honestly)

Prompts: "Tell me about a mistake."
Use a *real* mistake of yours from doing these exercises — e.g., your first trace of the auth flow missed the restricted-token projectId override ([workspace.ts#L268-L270](../../../apps/web/lib/auth/workspace.ts#L268-L270)) and your "bug report" was wrong; or your test for click recording was eaten by dedup and you initially blamed the code.
Structure: what you asserted → how reality corrected you → the *procedural* change (now you verify line numbers / write "hypothesis" labels / test your test first).
Senior-signal details: the fix is to your process, not just the instance; you tell it without flinching.
Resume bullet: (not resume material — interview-only.)

## Story 8: Influence without authority (maintainer interaction)

Prompts: "Working with people you don't control" / "Driving change across teams."
Source: actually filing the M2 proposal upstream using the [maintainer templates](../07-career-and-collaboration/03-maintainer-communication.md).
Situation: found a real gap (webhook cache miss = silent event loss) in a popular OSS project where you have zero standing.
Task: get maintainer agreement before writing code they'd have to maintain.
Action: issue with evidence (the TODO comment, the failure scenario), scoped proposal, explicit "what I'd commit to," accepted their direction on specifics.
Result: fill in honestly — even "maintainers deprioritized it; I kept my fork's patch and learned their triage criteria" is a strong, real answer.
Senior-signal details: you priced *their* maintenance cost into your proposal; evidence-first framing.
Resume bullet: *Contributed reliability analysis and fixes to Dub (20k+★ OSS link platform) through maintainer-collaborative proposals.*

---

Mapping table (prompt → story): conflict → 5; ambiguity → 1, 4; mistake → 7; technical tradeoff → 3, 4; leadership/influence → 6, 8; proudest work → 3 or 6; failure recovery → 7. Rehearse the top-left of this table twice as much — conflict and mistake questions appear in nearly every loop.
