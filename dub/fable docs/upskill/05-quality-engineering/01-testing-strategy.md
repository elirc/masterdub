# Testing Strategy

## What this repo actually does (and doesn't)

| Layer | Present? | Where | Notes |
| --- | --- | --- | --- |
| Unit tests | **Essentially absent** in apps/web | — | no unit suite for `processLink`, `withWorkspace`, cache logic |
| API integration tests | **The main suite** | [apps/web/tests/](../../../apps/web/tests/) (vitest) | real HTTP against a *deployed instance* via `E2E_BASE_URL` + `E2E_TOKEN` ([tests/utils/integration.ts#L22-L32](../../../apps/web/tests/utils/integration.ts#L22-L32)) |
| Browser E2E | Present | [apps/web/playwright/](../../../apps/web/playwright/) | coverage breadth unverified |
| Contract | Implicit | Zod schemas + [tests/utils/schema.ts](../../../apps/web/tests/utils/schema.ts) | tests assert responses against expected shapes (`expectedLink`) |

Config tells the story: [vitest.config.ts](../../../apps/web/vitest.config.ts) sets a **50-second** test timeout (network!), and the `test` script runs `-no-file-parallelism --bail=1` — sequential files, stop at first failure, because tests share one live workspace (`acme`, seeded resources in [tests/utils/resource.ts](../../../apps/web/tests/utils/resource.ts)) and can interfere.

## The tradeoff, stated fairly (this is an interview answer)

Testing against a live deployment buys: real auth, real DB, real caches, real Zod parsing — the *whole* stack per test, zero mock drift. It costs: slowness, flake exposure (network, shared state), no fast local red-green loop, inability to test failure injection (how do you make Redis fail in a deployed instance?), and CI needing a standing environment with secrets. That last cost is why the fan-out reliability code — the subtlest logic in the repo — has no direct tests: it's *unreachable* by black-box HTTP assertions.

The strategy makes sense for a public-API company: the API contract is the product, so contract-level tests give the most protection per test. The gap it leaves is exactly where you'd add unit tests: pure-ish domain functions (`processLink`), policy logic (scope intersection), cache-key normalization.

## What belongs at each layer (transferable)

- **Unit**: pure logic with branching (validation rules, permission math, key normalization). Fast, failure-injectable.
- **Integration (this repo's kind)**: contract shape, status codes, authz outcomes (cross-tenant 404s, permission 403s), happy-path persistence.
- **E2E (browser)**: critical user journeys only — login, create link, see analytics. Not for logic coverage.
- **Don't test**: framework behavior, generated code (Prisma client), third-party services' correctness — test your *use* of them.

## Isolation, fixtures, time, flake prevention — as practiced here

- Cleanup via `onTestFinished(() => h.deleteLink(id))` ([create-link.test.ts#L50-L56](../../../apps/web/tests/links/create-link.test.ts#L50-L56)) — create-your-own-data, delete-after; shared seeds (the E2E workspace/user/token) are read-only by convention.
- Randomized identifiers (`randomId`, `randomTagName` — [tests/utils/helpers.ts](../../../apps/web/tests/utils/helpers.ts)) prevent cross-run collisions on unique columns.
- `describe.sequential` ([create-link.test.ts#L16](../../../apps/web/tests/links/create-link.test.ts#L16)) where order matters.
- Time: expiry/analytics tests must cope with ingestion lag (Tinybird) — poll-with-timeout is the honest pattern; fixed sleeps are flake factories.
- A test-only hook lives in prod code: `NODE_ENV === "test"` adds a 5s webhook delay ([qstash.ts#L88](../../../apps/web/lib/webhook/qstash.ts#L88)) — note the smell and the pragmatism simultaneously.
- There's even a cleanup cron for e2e leftovers (`app/(ee)/api/cron/cleanup/e2e-tests`, referenced in the schema index comment [link.prisma#L104](../../../packages/prisma/schema/link.prisma#L104)) — when your tests hit a live system, *the system itself* needs a janitor. That detail delights interviewers.

## Drills

1. List three bugs the current suite **cannot catch** by construction (e.g., allSettled swallow, cache-shape drift on hit-path, token-cache revocation race) and which layer would.
2. Read [folder-link-access.test.ts](../../../apps/web/tests/links/folder-link-access.test.ts) and write down which rung of the [authz ladder](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) each assertion pins.
3. Sketch the vitest unit test file you'd write for `mapScopesToPermissions` intersection logic — no network, table-driven.

Interview angle: "how do you decide what to test?" — answer with the layer table + this repo's tradeoff, honestly stated. Contrarian-but-defensible positions backed by a real example read as senior. Cross-link: [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) review round 2.
