# Performance Thinking

## Rule zero: measure first

Every perf claim needs a number attached to a domain: **render** (React profiler), **network** (waterfall), **server** (route timing — the repo literally has `console.time("getAnalytics")` at [analytics route L119-L131](../../../apps/web/app/api/analytics/route.ts#L119-L131)), **DB** (query plans, slow log), **bundle** (next build output), **cache** (hit ratios — the HIT/MISS logs in [cache.ts#L76-L94](../../../apps/web/lib/api/links/cache.ts#L76-L94) are a free hit-ratio source), **memory** (instance limits).

## The performance domains of this repo, with likely hotspots

| Domain | Hot spot | Evidence / anchor | What to check |
| --- | --- | --- | --- |
| Redirect latency | cache-miss path: Redis timeout → Vercel cache → PlanetScale query | [cache.ts#L66-L100](../../../apps/web/lib/api/links/cache.ts#L66-L100), [link.ts#L85-L105](../../../apps/web/lib/middleware/link.ts#L85-L105) | tail latency on misses; password links do an *extra* DB read per hit ([link.ts#L197](../../../apps/web/lib/middleware/link.ts#L197)) |
| Click pipeline | per-click MySQL reads for webhook-enabled links | usage check [record-click.ts#L266-L279](../../../apps/web/lib/tinybird/record-click.ts#L266-L279) + link+tags join [#L324-L348](../../../apps/web/lib/tinybird/record-click.ts#L324-L348) | it's off the response path (waitUntil) but still DB load ∝ clicks |
| Links list | fuzzy search on large workspaces | mode flip at [links route L34-L36](../../../apps/web/app/api/links/route.ts#L34-L36) — the codebase *already* encodes "fuzzy is too slow at scale" | whether search uses the composite index; `MEGA_WORKSPACE_LINKS_LIMIT` value |
| Analytics | Tinybird pipe latency; date-range × groupBy combinations | `getAnalytics` timing | plan-gated ranges bound worst cases ([route L103-L109](../../../apps/web/app/api/analytics/route.ts#L103-L109)) — limits as perf armor |
| Dashboard render | 100-row link lists with per-card hooks | [link-card.tsx](../../../apps/web/ui/links/link-card.tsx) — memo'd cards, conditional folder fetch, prefetch-on-visible | N cards × `useFolder` — SWR dedupes identical folder keys; verify when folders differ (N distinct fetches — an N+1 in HTTP clothing, mitigated by `enabled`) |
| Bundle | dashboard-wide client components, tiptap/react-pdf deps | [apps/web/package.json](../../../apps/web/package.json) heavyweights | dynamic imports for editors/PDF; check with `next build` analyzer |

## Finding the classics, repo-specifically

- **N+1 queries (server)**: the repo's antidote is includes at query time (`includeTags` — [create-link.ts#L132-L136](../../../apps/web/lib/api/links/create-link.ts#L132-L136)) and JSON aggregation in raw SQL ([record-click.ts#L326-L341](../../../apps/web/lib/tinybird/record-click.ts#L326-L341)). Audit: any `for` loop containing `await prisma.` is a suspect — grep and triage.
- **Serial async**: consecutive independent `await`s. The fan-outs are already parallel; look in less-loved code (cron jobs, importers under [app/(ee)/api/cron/import](../../../apps/web/app/(ee)/api/cron/import)).
- **Missing indexes**: every index in [link.prisma#L95-L105](../../../packages/prisma/schema/link.prisma#L95-L105) names its query — so a *new* query pattern (new filter combination) needs an index conversation, not just code.
- **Unbounded queries**: list endpoints paginate (`LINKS_MAX_PAGE_SIZE = 100` in the zod schema, [links.ts#L36](../../../apps/web/lib/zod/schemas/links.ts#L36)); the risk is internal paths — bulk jobs, exports ([lib/api/create-downloadable-export.ts](../../../apps/web/lib/api/create-downloadable-export.ts)) — where nobody parses a limit.
- **Expensive renders**: filter-change → new SWR key → full list replace; check for layout thrash and unvirtualized long lists.

## The senior habit: capacity math before code

Example you can do now: webhook-enabled link at 100 clicks/sec → 100 MySQL point-reads/sec from the usage check + 100 join queries/sec for payload assembly. Is that fine? (Probably, for a while — indexed point reads.) At what multiple does it stop being fine, and what's the fix (cache usage row with 10s TTL; denormalize tags into the webhook cache)? Interviewers reward this arithmetic far more than reciting "use Redis."

Drill: pick the links-list endpoint. Write down its work per request (queries, their indexes, payload size) for a 50-link workspace and a 500k-link MEGA workspace. Find every place the code already branches on scale — then propose the *next* guardrail it will need.

Interview angle: "the site is slow — go" → structure your answer by the domain table above (isolate the domain with measurements, then optimize inside it). Cross-link: [08/02 Q9](../08-interview-prep/02-frontend-framework-questions.md), [08/04 variation 2](../08-interview-prep/04-system-design-from-this-repo.md).
