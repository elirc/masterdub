# Data Model and Persistence

## The stores and their jobs

| Store | Job | Access path |
| --- | --- | --- |
| MySQL (PlanetScale) | entities, relationships, denormalized counters | Prisma ([packages/prisma](../../../packages/prisma)); raw HTTP driver on hot paths ([lib/planetscale](../../../apps/web/lib/planetscale/)) |
| Tinybird | append-only event history (clicks/leads/sales), analytics queries | HTTP ingest ([record-click.ts#L179-L189](../../../apps/web/lib/tinybird/record-click.ts#L179-L189)); pipes in [packages/tinybird](../../../packages/tinybird) |
| Redis (Upstash) | link cache, token cache, dedup, rate limits, streams | [lib/upstash](../../../apps/web/lib/upstash/) |
| R2 | uploaded images/assets | [lib/storage.ts](../../../apps/web/lib/storage.ts) |

## Key entities (from 36 schema files in [packages/prisma/schema/](../../../packages/prisma/schema/))

`Project` (=workspace, the tenant) 1—n `Link`; `Link` n—n `Tag` (via `LinkTag`); `Project` 1—n `Domain`; `User` n—n `Project` (via `ProjectUsers` with `role`); partner side: `Program` (per workspace) —< `ProgramEnrollment` >— `Partner`, and `Link` optionally carries `(programId, partnerId)` tying a link to an enrollment ([link.prisma#L83-L89](../../../packages/prisma/schema/link.prisma#L83-L89)). `Customer` and `Commission` hang off links for conversion/payout tracking.

## Reading the Link model like a senior ([link.prisma](../../../packages/prisma/schema/link.prisma))

- **Uniques are lookups and idempotency keys**: `@@unique([domain, key])` (L95) — the redirect lookup *and* the collision guard that makes create-retries safe; `@@unique([projectId, externalId])` (L96) — customer-supplied idempotency for API users; `shortLink @unique` (L6) — the same fact as (domain,key) in a second representation (kept consistent by construction at [create-link.ts#L61](../../../apps/web/lib/api/links/create-link.ts#L61)).
- **Every index has a comment naming its query** (L95-L105) — e.g. `@@index([projectId, folderId, archived, createdAt(sort: Desc)])` "most getLinksForWorkspace queries". This discipline is copyable to any repo: *an index without a named query is a guess*.
- **Denormalized counters** (`clicks`, `leads`, `sales`, `saleAmount BigInt` — cents!) with `lastClicked` timestamps: read models maintained by the click pipeline, not by triggers. Consistency: eventual, unreconciled (possible risk — see [risk register](../09-reference/risk-register.md)).
- **State as timestamps**: `expiresAt`, `disabledAt`, `testStartedAt/CompletedAt` — nullable datetimes double as booleans+history. Cheaper than status enums until you need more than two states.
- **Money as BigInt cents** (L61) — never floats. Interview-quotable.

## Transactions and consistency expectations

- A single `prisma.link.create` with nested writes (tags, webhooks, dashboard — [create-link.ts#L56-L138](../../../apps/web/lib/api/links/create-link.ts#L56-L138)) is atomic in MySQL. **Everything after it is not** — cache, Tinybird, R2, webhooks are `waitUntil`+`allSettled`, i.e., best-effort eventual.
- Cross-store invariants are therefore *converging*, not guaranteed: Redis has a 24h TTL backstop; Tinybird link metadata is re-recorded on changes; counters can drift. The design leans on **TTL + re-derivability** instead of transactions — a legitimate strategy you should be able to name and defend.
- PlanetScale specifics: no foreign-key constraints traditionally (Prisma emulates relations — verify `relationMode` in the schema before asserting); raw driver used where connection pooling matters ([record-click.ts#L195](../../../apps/web/lib/tinybird/record-click.ts#L195)).

## How to change the schema safely here

1. Edit the `.prisma` file; run `pnpm prisma:format` then `pnpm prisma:generate` (types) — both __inferred__ from [apps/web/package.json](../../../apps/web/package.json).
2. `pnpm prisma:push` applies schema to the dev DB (db-push workflow — **inferred**; no migrations dir in the package. For prod, PlanetScale-style branch/deploy-request flow is likely — verify with maintainers before claiming).
3. Expand-migrate-contract for anything breaking: add nullable column → backfill → make required/drop old. Never rename in place on a live table.
4. Grep for raw SQL touching the table (`conn.execute` sites) — Prisma won't tell you about those. `Link` has several ([record-click.ts#L196-L199](../../../apps/web/lib/tinybird/record-click.ts#L196-L199), [#L324-L341](../../../apps/web/lib/tinybird/record-click.ts#L324-L341)).
5. Check the Redis cached shape (`formatRedisLink`) and Tinybird `recordLink` — three stores may need the new field.

What a junior misses: steps 4–5 — the schema's consumers beyond Prisma. What a senior checks first: which uniques/indexes the change invalidates, and the rollback story (**rollback** = can you revert the deploy without reverting the data?).

Interview angle: schema design + migrations questions → [08/03 Q3, Q7, Q8](../08-interview-prep/03-api-and-data-modeling-questions.md). The index-comment discipline and BigInt-cents are memorable concrete details to cite.

Drill: design the schema change to add per-link `clickLimit` (max clicks then disable). Write: column definition, which code paths must read it (redirect hot path — cached shape!), the backfill, and the rollback. Compare your redirect-path plan against how `expiresAt` flows from schema → `formatRedisLink` → [link.ts#L234-L252](../../../apps/web/lib/middleware/link.ts#L234-L252).
