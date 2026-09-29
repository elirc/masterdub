# Dub Upskill Curriculum

A training lab built on the real [Dub](https://dub.co) codebase for one learner profile: a **junior fullstack JS engineer** (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page pairs two things:

1. **Codebase-specific knowledge** — where things live in *this* repo, how its real flows work, with exact file/line anchors.
2. **Transferable skill** — why the pattern exists, what the alternatives are, how it fails, and how to talk about it in an interview.

## What this repo is

Dub is an open-source **link management and partner/affiliate attribution platform**: short links with custom domains, click/lead/sale conversion analytics, A/B testing, geo/device targeting, webhooks, and a full partner-program product. Technically it is a **pnpm + Turborepo monorepo** ([pnpm-workspace.yaml](../../pnpm-workspace.yaml), [turbo.json](../../turbo.json)) whose main product is one large Next.js App Router application ([apps/web](../../apps/web)) plus shared packages: Prisma schema/client ([packages/prisma](../../packages/prisma)), UI kit ([packages/ui](../../packages/ui)), utils, email templates, embeds, and a CLI. Persistence is MySQL (PlanetScale) through Prisma; high-volume click events go to **Tinybird** (a columnar analytics store); hot-path caching and background jobs run on **Upstash Redis and QStash**. The redirect hot path is Next.js middleware ([apps/web/middleware.ts](../../apps/web/middleware.ts)) that routes by hostname and serves billions of redirects — which is why this codebase is unusually good at teaching caching, async side effects, and multi-tenant authorization. Parts of the app under `(ee)` directories (e.g. [apps/web/app/(ee)](../../apps/web/app/(ee))) are the commercial/enterprise surface (partner programs, payouts, conversion tracking).

## The learning tracks

| Module | What it builds |
| --- | --- |
| [00-fast-track.md](00-fast-track.md) | A weekend: run it, trace two flows, make one safe change |
| [01-codebase-cartography](01-codebase-cartography/README.md) | Map: system shape, reading order, glossary, key flows |
| [02-stack-and-language-mastery](02-stack-and-language-mastery/README.md) | JS/TS/Node/React/Next mental models, anchored to real files |
| [03-architecture-and-patterns](03-architecture-and-patterns/README.md) | Boundaries, data model, authz, async reliability, pattern catalog, critique |
| [04-code-reading-gym](04-code-reading-gym/README.md) | Annotation drills, trace tables, fake-code contrasts, review katas |
| [05-quality-engineering](05-quality-engineering/README.md) | Testing, debugging, performance, security, observability |
| [06-contribution-practice](06-contribution-practice/README.md) | Junior tickets → mid-level features → senior projects |
| [07-career-and-collaboration](07-career-and-collaboration/README.md) | Review mindset, PRs/RFCs, maintainer communication |
| [08-interview-prep](08-interview-prep/README.md) | 45+ question cards, system design from this repo, STAR stories, cram plan |
| [09-reference](09-reference/) | Command cheatsheet, risk register, rubrics, verification log |

## Recommended paths

- **Brand-new junior** — 00-fast-track → 01 (all) → 02 (all) → 04-code-reading-gym drills → 06/01-good-first-tickets. Eight weeks: add 03 and 05, one mid-level ticket.
- **Junior with stack familiarity (the target learner)** — 00-fast-track → 01/05-key-flows → 03 (all) → 04 drills → 05/01–03 → one ticket from 06/02 → 08-interview-prep in parallel from week 2.
- **Mid-level new to the repo** — 01/01-system-map + 01/05-key-flows → 03/05-pattern-catalog → 03/06-architecture-critique → 06/03-senior-build-projects.
- **Senior doing architecture review** — 03/06-architecture-critique → [09-reference/risk-register.md](09-reference/risk-register.md) → 03/04-side-effects-async-and-reliability.
- **Candidate with an interview in two weeks** — go straight to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md); it schedules everything else you need.

## Conventions in these docs

- **File anchors**: `[file.ts](relative/path#L10-L20)` or `path:10-20`. Line numbers were confirmed against the working tree on the date in the [verification log](09-reference/verification-log.md). If the repo has moved since, search for the quoted identifier instead of trusting the number.
- **Fake code**: any snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Everything else is real.
- **Drills** end with self-grading criteria (Basic / Solid / Strong). Grade yourself honestly; the gap between Solid and Strong is exactly the gap between junior and mid-level.
- **Verified vs inferred**: commands marked __verified__ were actually run during authoring; __inferred__ were read from manifests but not executed (most need env vars/services this machine doesn't have).
- **Uncertainty**: claims we could not fully confirm are labeled "investigate" or "possible risk" — never presented as bugs.

## The mindset ladder

- **Junior asks:** "How do I make it work?"
- **Mid-level asks:** "Is this the right pattern? What are the alternatives and tradeoffs?"
- **Senior asks:** "What does this commit us to? Who pays the cost when it fails? How do we reduce the blast radius?"

Interviews for mid-level roles test exactly the second and third questions. When an interviewer asks "how would you build link shortening?", they don't want a definition of a hash function — they want you to talk about cache invalidation ([lib/api/links/cache.ts](../../apps/web/lib/api/links/cache.ts)), what happens when Redis is down, and who is authorized to see whose analytics. This codebase gives you concrete, defensible answers to all of that. Use it.
