# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — happy path, obvious errors. (Table stakes; juniors stop here.)
2. **Is it correct?** — edge cases, concurrency, failure paths. *In this repo: what happens on cache hit vs miss; what happens when the `waitUntil` work fails; what happens when it runs twice.*
3. **Will it stay correct?** — tests pinning the contract; invariants documented; no implicit cross-file dependencies added (the [cache-shape trap](../03-architecture-and-patterns/01-boundaries-and-layers.md)).
4. **Does it fit?** — follows the repo's patterns (scoped lookups, error taxonomy, zod-at-boundary, `requiredPermissions` declared) rather than inventing parallel ones.
5. **Is it kind to future maintainers?** — names, comments where the code can't speak (cross-file contracts), no gratuitous cleverness.

Blocking findings live in layers 1–3 (and security anywhere). Layer 4–5 findings are important/optional — don't block on taste.

## The repo-specific reviewer checklist

- [ ] New DB reads: tenant-scoped? Correct index exists for the new query shape ([link.prisma#L95-L105](../../../packages/prisma/schema/link.prisma#L95-L105) discipline)?
- [ ] New route: `requiredPermissions` declared? Zod schemas for body *and* response? Error codes from the taxonomy?
- [ ] Touches the redirect path or cached link shape: `formatRedisLink` updated? Invalidation on every mutation path?
- [ ] New side effect: inside or outside `waitUntil` — argued, not defaulted? Idempotent under retry? Visible when it fails?
- [ ] Public contract change (API/webhook/published package): backwards compatible or shimmed + telemetered?
- [ ] Tests: does at least one assertion fail if the *interesting* part regresses (not just 200-status theater)?
- [ ] Security checklist rows triggered? ([05/05](../05-quality-engineering/05-security-checklist.md))

## Comment craft — the same finding, three ways

Bad (verdict without reasoning): *"This is wrong, use getLinkOrThrow."*
Better (reason attached): *"This `findUnique` by id isn't workspace-scoped, so any authenticated user can read other tenants' links."*
Best (reason + path + tone that assumes good faith):
> This lookup takes the id from the request but doesn't scope by workspace — as written, a user from workspace A can read workspace B's links by guessing ids. `getLinkOrThrow({ workspaceId, linkId })` handles this (and the 404-vs-403 behavior) — same pattern as the analytics route. Happy to pair if the calling context makes that awkward.

Severity words matter: prefix findings with **blocking:** / **important:** / **nit:** so authors can triage. Ask at least one *genuine* question per review — "what led you to X?" regularly reveals a constraint that changes your finding.

## Receiving review (half the skill)

Respond to every comment (fix, push back with reasons, or open a follow-up ticket — never silence). Push back with evidence, not seniority: "I kept it settled rather than sequential because these writes are independent — see the pattern at record-click.ts#L179" wins; "it's fine" doesn't. Thank reviewers for catches that would have burned you; it compounds.

Drill: review katas [1, 5, and 7](../04-code-reading-gym/04-review-katas.md) again, but this time write the *full review text* as you'd post it — severity prefixes, one genuine question, and a closing summary line. Compare tone against the "Best" example above.

Interview angle: "walk me through how you review a PR" — recite the five layers with one repo-concrete example per layer. "Tell me about pushing back in review" → build the STAR from a kata write-up ([08/06 story 5](../08-interview-prep/06-behavioral-star-stories.md)).
