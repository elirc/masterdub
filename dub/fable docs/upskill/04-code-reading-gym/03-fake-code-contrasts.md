# Fake-Code Contrasts

Eight bad-vs-better pairs. All snippets are **illustrative fake code, not from this repo** — each pair is tied to the real pattern the repo uses. The "bad" version is a genuine novice instinct, not a strawman: you have probably written each one.

---

## Contrast 1: Missing tenant scope (IDOR)

```ts
// Illustrative fake code: not from this repo — BAD
export const GET = withWorkspace(async ({ searchParams }) => {
  const link = await prisma.link.findUnique({ where: { id: searchParams.linkId } });
  return NextResponse.json(link); // any authenticated user can read ANY link by id
});
```

```ts
// Illustrative fake code: not from this repo — BETTER
export const GET = withWorkspace(async ({ searchParams, workspace }) => {
  const link = await getLinkOrThrow({ workspaceId: workspace.id, linkId: searchParams.linkId });
  return NextResponse.json(link);
});
```

Real anchor: [get-link-or-throw.ts](../../../apps/web/lib/api/links/get-link-or-throw.ts) used at [analytics route L72-L78](../../../apps/web/app/api/analytics/route.ts#L72-L78). Being *authenticated* to workspace A says nothing about link X — the lookup itself must carry the tenant.

## Contrast 2: Read-modify-write counter

```ts
// Illustrative fake code: not from this repo — BAD
const link = await prisma.link.findUnique({ where: { id } });
await prisma.link.update({ where: { id }, data: { clicks: link.clicks + 1 } }); // lost updates under concurrency
```

```ts
// Illustrative fake code: not from this repo — BETTER
await prisma.link.update({ where: { id }, data: { clicks: { increment: 1 } } });
// or, hot path: UPDATE Link SET clicks = clicks + 1 WHERE id = ?
```

Real anchor: [record-click.ts#L196-L199](../../../apps/web/lib/tinybird/record-click.ts#L196-L199). Push arithmetic into the store; the row lock is the synchronization.

## Contrast 3: Blocking the response on bookkeeping

```ts
// Illustrative fake code: not from this repo — BAD
export async function redirectHandler(req) {
  const link = await lookup(req);
  await recordAnalytics(req, link);   // user waits for Tinybird
  await updateCounters(link.id);      // ...and MySQL
  return NextResponse.redirect(link.url);
}
```

```ts
// Illustrative fake code: not from this repo — BETTER
export async function redirectHandler(req, ev) {
  const link = await lookup(req);
  ev.waitUntil(recordAnalytics(req, link)); // fire-and-forget, platform-managed
  return NextResponse.redirect(link.url);
}
```

Real anchor: [link.ts#L553-L567](../../../apps/web/lib/middleware/link.ts#L553-L567). The follow-up question you must be ready for: what's the failure story of the background work? (See [pattern 5](../03-architecture-and-patterns/05-pattern-catalog.md).)

## Contrast 4: Swallowed errors

```ts
// Illustrative fake code: not from this repo — BAD
try {
  await Promise.all([ingestEvent(data), bumpCounter(id), setCache(key)]);
} catch (e) {
  // TODO handle later
}
```

```ts
// Illustrative fake code: not from this repo — BETTER
const ops = [
  { name: "ingest", run: () => ingestEvent(data) },
  { name: "counter", run: () => bumpCounter(id) },
  { name: "cache", run: () => setCache(key) },
];
const results = await Promise.allSettled(ops.map((o) => o.run()));
results.forEach((r, i) => {
  if (r.status === "rejected") log.error("side-effect failed", { op: ops[i].name, reason: r.reason });
});
```

Real anchor: [record-click.ts#L233-L262](../../../apps/web/lib/tinybird/record-click.ts#L233-L262) (which does the logging, with the index-fragility caveat covered in [ticket 2](../06-contribution-practice/01-good-first-tickets.md)). `all`+empty-catch loses *which* effect failed and lets siblings' outcomes vanish.

## Contrast 5: Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo — BAD
// component renders prisma rows directly; DB rename = UI break; leaks internal fields
const { data } = useSWR<PrismaLink[]>(`/api/links-raw`);
return <div>{data[0].projectId} {data[0].shortLink}</div>;
```

```ts
// Illustrative fake code: not from this repo — BETTER
// API returns a transformed, schema-parsed shape; UI types come from the contract
const { data } = useSWR<ExpandedLinkProps[]>(key);  // z.infer of the response schema
```

Real anchor: `transformLink` ([lib/api/links/utils/transform-link.ts](../../../apps/web/lib/api/links/utils/transform-link.ts)) converts rows to API shape at [create-link.ts#L230-L231](../../../apps/web/lib/api/links/create-link.ts#L230-L231); the workspace's `workspacePreferences` are explicitly *hidden from API responses* at [workspace.ts#L355](../../../apps/web/lib/auth/workspace.ts#L355) — shape control is security, not just tidiness.

## Contrast 6: Cache key that lies

```ts
// Illustrative fake code: not from this repo — BAD
await redis.set(`link:${domain}:${key}`, link);          // write path: raw key
const hit = await redis.get(`link:${domain}:${key.toLowerCase()}`); // read path: lowercased
// works for lowercase links; MixedCase links never hit cache — or worse, hit the wrong entry
```

```ts
// Illustrative fake code: not from this repo — BETTER
const cacheKey = createKey({ domain, key }); // ONE normalization function, used by both paths
```

Real anchor: the repo centralizes this in `_createKey`/`decodeKey` with per-domain case sensitivity ([cache.ts#L66-L70](../../../apps/web/lib/api/links/cache.ts#L66-L70), [case-sensitivity.ts](../../../apps/web/lib/api/links/case-sensitivity.ts)) — and the comment on the read path explains a subtle exception. Normalization must be a shared function, never inline.

## Contrast 7: Any-typed boundary

```ts
// Illustrative fake code: not from this repo — BAD
const body: any = await req.json();
if (body.url) createLink(body); // typos, wrong types, extra fields all flow in
```

```ts
// Illustrative fake code: not from this repo — BETTER
const body = createLinkBodySchema.parse(await req.json()); // typed, validated, documented in one move
```

Real anchor: [links route L54-L56](../../../apps/web/app/api/links/route.ts#L54-L56). Also note the repo's `parseRequestBody` wrapper handles malformed JSON into a clean 400 ([lib/api/utils.ts](../../../apps/web/lib/api/utils.ts) — verify) instead of an unhandled SyntaxError.

## Contrast 8: Casual public-contract change

```ts
// Illustrative fake code: not from this repo — BAD (PR titled "clean up analytics params")
- const { eventType, endpoint } = analyticsPathParamsSchema.parse(params);
+ const { event } = newCleanSchema.parse(params); // old param names now 422 — every existing API client breaks
```

```ts
// Illustrative fake code: not from this repo — BETTER
const parsed = newCleanSchema.or(legacySchema.transform(migrateLegacy)).parse(params);
metrics.increment("analytics.legacy_params", { used: isLegacy }); // measure before ever removing
```

Real anchor: the deliberate back-compat shims at [analytics route L27-L31](../../../apps/web/app/api/analytics/route.ts#L27-L31) and deprecated field aliases at [track/lead route L17-L21](../../../apps/web/app/(ee)/api/track/lead/route.ts#L17-L21). Public contracts are append-only until telemetry proves otherwise.

---

Drill: for each contrast, name the *category* (tenancy / concurrency / latency / observability / coupling / consistency / boundary typing / contract stability) without looking, then find one more real instance of the good pattern in the repo. Solid = 6/8 categories named; Strong = 8/8 plus the extra instances.
