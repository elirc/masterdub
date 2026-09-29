# 08 — Interview Prep

Target: **mid-level fullstack JS interviews**. This module turns the rest of the curriculum into interview performance. Over 60% of the question cards are anchored to real Dub code, because the strongest thing a mid-level candidate can do is answer with *concrete evidence*: a real file, a real tradeoff, a real failure mode.

## How mid-level fullstack loops are typically structured

1. **Screen** (30–45 min): JS/TS fundamentals + a small coding exercise. Feed: [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md).
2. **Technical deep-dive** (60 min): "tell me about a system you know well" — this is where studying Dub *is* the preparation; you know a system doing billions of redirects.
3. **Practical coding** (60–90 min): build/extend a feature; increasingly a debugging round instead. Feed: [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md).
4. **System design** (45–60 min): feed: [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) — you will likely be asked the literal system this repo implements.
5. **Behavioral** (45 min): feed: [06-behavioral-star-stories.md](06-behavioral-star-stories.md).

## The golden rule

Answer with **example → tradeoff → failure mode**, never with a definition. "What's caching?" — weak answers define it; strong answers say: "In a link shortener I've studied, redirects hit a 5-second in-process LRU in front of Redis in front of MySQL; the LRU exists to absorb viral-link stampedes; the failure mode is per-instance staleness, bounded by the TTL." Same knowledge, hire/no-hire difference.

Use this repo as a **portfolio of talking points**, and be honest about provenance: "in an open-source codebase I've contributed to / studied deeply" is credible and verifiable — and if you complete even three tickets from [06-contribution-practice](../06-contribution-practice/README.md), it's simply true.

## Contents

| File | Cards | Repo-anchored |
| --- | --- | --- |
| [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) | 14 | 13/14 |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | 10 | 9/10 |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | 12 | 12/12 |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | 1 full walkthrough + 5 variations | all |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | 4 debugging + 3 review sims | all |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | 8 worksheets | all |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | the schedule | — |
