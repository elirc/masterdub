# Trace Tables

Fill each table yourself first (columns: step, file/line, value shape, owner, transformation, risk), then compare. Completed reference rows are provided for Trace 1; the rest give you the skeleton and checkpoints — the work is yours.

## Trace 1 (reference, UI → API): the `search` filter, from keystroke to SQL

| Step | File | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | links filter UI (`ui/links/use-link-filters.tsx`) | `"onboarding"` string in an input | client | debounced into router querystring | none |
| 2 | [use-links.ts#L29-L50](../../../apps/web/lib/swr/use-links.ts#L29-L50) | `?search=onboarding&workspaceId=ws_…` | client | serialized into the SWR key | key explosion per keystroke if debounce fails |
| 3 | [links route GET L22](../../../apps/web/app/api/links/route.ts#L22) | `searchParams` record | server | `getLinksQuerySchemaExtended.parse` → typed filters | Zod rejects → 422, UI must handle |
| 4 | [route L24-L28](../../../apps/web/app/api/links/route.ts#L24-L28) | filters + folderIds | server | `validateLinksQueryFilters` adds folder authz | missing here = folder data leak |
| 5 | [route L30-L36](../../../apps/web/app/api/links/route.ts#L30-L36) | + `searchMode: exact\|fuzzy` | server | mega-workspace check flips search strategy | fuzzy on huge tenants = slow query |
| 6 | [get-links-for-workspace.ts](../../../apps/web/lib/api/links/get-links-for-workspace.ts) | Prisma `where` | domain | filters → SQL against `@@index([projectId, folderId, archived, createdAt])` | search likely bypasses that index (verify) |

Checkpoint question: at which single step could tenant isolation be lost, and what guards it? (Step 6 — `workspaceId` is a required argument.)

## Trace 2 (persistence): `expiresAt` from API payload to a user seeing the expired page

Skeleton: `POST /api/links {expiresAt: "2026-01-01"}` → parseDateSchema in [links.ts zod](../../../apps/web/lib/zod/schemas/links.ts) → `processLink` ([process-link.ts#L77](../../../apps/web/lib/api/links/process-link.ts#L77) — note it also accepts natural language via `parseDateTime`, verify) → `new Date(expiresAt)` at [create-link.ts#L71](../../../apps/web/lib/api/links/create-link.ts#L71) → MySQL `DateTime?` ([link.prisma#L8](../../../packages/prisma/schema/link.prisma#L8)) → `formatRedisLink` (does it keep expiresAt? the schema comment says "stored on Redis via ttl" — reconcile that with the runtime check!) → redirect compare [link.ts#L234](../../../apps/web/lib/middleware/link.ts#L234).
Checkpoints: (a) how many timezone/format conversions happen? (b) the comment claims Redis TTL handles expiry, the middleware checks `expiresAt` explicitly — which is actually load-bearing, and could they disagree? (c) risk column for step "cached link edited to remove expiry."

## Trace 3 (auth): a `dub_` restricted token, from header to `permissions` array

Skeleton: `Authorization: Bearer dub_xxx` → prefix detection [workspace.ts#L120](../../../apps/web/lib/auth/workspace.ts#L120) → `hashToken` → cache/DB lookup [#L175-L212](../../../apps/web/lib/auth/workspace.ts#L175-L212) → expiry gate → `token.projectId` overrides workspace hint [#L268-L270](../../../apps/web/lib/auth/workspace.ts#L268-L270) → membership fetch [#L342-L359](../../../apps/web/lib/auth/workspace.ts#L342-L359) → role→permissions [#L410](../../../apps/web/lib/auth/workspace.ts#L410) → scope intersection [#L413-L418](../../../apps/web/lib/auth/workspace.ts#L413-L418) → `throwIfNoAccess`.
Checkpoints: (a) value shape of `permissions` at each step; (b) which two stores are consulted and in what order; (c) the risk row for "user's role downgraded five minutes ago" (token cache TTL vs role read — note the role is read *fresh* from the membership join, so which parts are actually cached?).

## Trace 4 (error): a duplicate `(domain, key)` from Prisma to the client toast

Skeleton: `prisma.link.create` throws P2002 → **not** P2025, so [handleApiError](../../../apps/web/lib/api/errors.ts#L92-L130) — trace which branch actually catches it. Careful: the route wraps `createLink` in try/catch and rethrows as `unprocessable_entity` ([links route L100-L105](../../../apps/web/app/api/links/route.ts#L100-L105)) — so the Prisma branch in `handleApiError` may never see it on this path. → JSON error body shape ([errors.ts#L23-L38](../../../apps/web/lib/api/errors.ts#L23-L38)) → SWR/fetch error → toast.
Checkpoints: (a) what `message` does the user actually read — is it helpful for a key collision? (b) where would you add the friendly "that key is taken" message, and why is `processLink`'s key check ([utils/keyChecks](../../../apps/web/lib/api/links/utils) — verify) supposed to catch this *before* the DB does? (c) so when does the race still reach Prisma? (Two concurrent creates of the same key — check-then-create TOCTOU.)

## Trace 5 (async): one click's `clickId`, across four storage locations

Skeleton: generated `nanoid(16)` or recovered ([link.ts#L254-L272](../../../apps/web/lib/middleware/link.ts#L254-L272)) → response cookie `dub_id_<domain>_<key>` ([#L274-L279](../../../apps/web/lib/middleware/link.ts#L274-L279)) → `clickIdCache:` Redis 5-min blob ([record-click.ts#L171-L175](../../../apps/web/lib/tinybird/record-click.ts#L171-L175)) → `recordClickCache` 1-hour dedup entry ([#L192](../../../apps/web/lib/tinybird/record-click.ts#L192)) → Tinybird `click_id` column → later `/track/lead` resolves it (find where: does it read the cookie param, the Redis blob, or query Tinybird? read [lib/api/conversions/track-lead.ts](../../../apps/web/lib/api/conversions/track-lead.ts)).
Checkpoints: (a) TTL of each location; (b) which locations agree by construction and which can diverge; (c) the conversion that arrives at minute 6 — which lookup saves it and what latency does it pay?

## Self-grading (all traces)

Basic: correct sequence of files. Solid: correct value *shapes* at each step and at least one transformation named precisely (e.g., "string → Date → DATETIME(3)"). Strong: every risk cell filled with a concrete failure, plus one checkpoint question answered with an anchor you found yourself.
