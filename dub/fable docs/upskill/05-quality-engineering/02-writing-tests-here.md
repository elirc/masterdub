# Writing Tests Here

Six recipes using the repo's actual harness. Commands are __inferred__ (need `E2E_BASE_URL`/`E2E_TOKEN` in `apps/web/.env`):

```bash
cd apps/web
pnpm test                              # whole suite, sequential, bail on first failure
pnpm test tests/links/create-link.test.ts   # one file (vitest positional filter)
```

The harness in one paragraph: `new IntegrationHarness()` + `await h.init()` gives you `http` (authed [HttpClient](../../../apps/web/tests/utils/http.ts) with the E2E bearer token), the seeded `workspace` (id, slug `acme`), `user`, and cleanup helpers (`deleteLink`, `deleteTag`, …) — [tests/utils/integration.ts#L15-L66](../../../apps/web/tests/utils/integration.ts#L15-L66). Response shapes are asserted against shared expected objects like `expectedLink` ([tests/utils/schema.ts](../../../apps/web/tests/utils/schema.ts)).

## Recipe 1: Happy-path create (the canonical shape)

Copy the pattern at [create-link.test.ts#L50-L80](../../../apps/web/tests/links/create-link.test.ts#L50-L80): register cleanup **before** the action (`onTestFinished` first, so cleanup runs even on assertion failure), act via `http.post`, assert `status` and `toStrictEqual({...expectedLink, ...overrides})`. `toStrictEqual` + spread-with-overrides pins the *entire* contract — new fields in the response fail the test, which is the point.

## Recipe 2: Validation failure (422)

```ts
// Illustrative fake code: not from this repo — follows repo patterns
test("rejects invalid destination URL", async () => {
  const { status, data } = await http.post<{ error: { code: string } }>({
    path: "/links",
    body: { url: "not-a-url", domain: "dub.sh" },
  });
  expect(status).toEqual(422);
  expect(data.error.code).toEqual("unprocessable_entity");
});
```

Assert the error **code**, not the message (messages are UX, codes are contract — [errors.ts](../../../apps/web/lib/api/errors.ts)). Existing examples: [create-link-error.test.ts](../../../apps/web/tests/links/create-link-error.test.ts).

## Recipe 3: Permission failure

Needs a second credential with narrower scope. Check `tests/utils/` for how token variants are provisioned (the harness carries one admin-ish token; a scoped-token fixture may need creating via `POST /tokens` with limited scopes, then cleanup). Assert 403 + code `forbidden` on e.g. `POST /links` with a read-only token. If provisioning proves impossible black-box, that's a finding — record it and unit-test `throwIfNoAccess` instead.

## Recipe 4: Cross-tenant rejection (the IDOR test)

Create a link normally, then request it with a *different* workspaceId query param: expect 404 (not 403 — anti-enumeration, [workspace.ts#L386-L389](../../../apps/web/lib/auth/workspace.ts#L386-L389)). Note the subtlety from [trace 3](../04-code-reading-gym/02-trace-tables.md): with a `dub_` token, the token's own projectId wins — so this test needs the *workspace mismatch* to happen at the resource lookup, e.g. `GET /links/[id]` for a link id from another (seeded) workspace. Verify what second-workspace fixtures exist in [tests/utils/resource.ts](../../../apps/web/tests/utils/resource.ts) before writing.

## Recipe 5: Async side effect (webhook / analytics)

Fire the action, then **poll** the observable outcome with a deadline:

```ts
// Illustrative fake code: not from this repo — follows repo patterns
async function eventually<T>(fn: () => Promise<T>, pred: (t: T) => boolean, ms = 30_000) {
  const deadline = Date.now() + ms;
  while (Date.now() < deadline) {
    const v = await fn();
    if (pred(v)) return v;
    await new Promise((r) => setTimeout(r, 2_000));
  }
  throw new Error("condition not met in time");
}
// click the link, then:
await eventually(
  () => http.get({ path: `/analytics`, query: { workspaceId, linkId, event: "clicks" } }),
  (res) => res.data.clicks >= 1,
);
```

Remember the recording guards will eat naive attempts: bot-looking UAs are dropped ([record-click.ts#L75-L86](../../../apps/web/lib/tinybird/record-click.ts#L75-L86)) and repeat clicks dedupe for an hour ([#L88-L108](../../../apps/web/lib/tinybird/record-click.ts#L88-L108)). Send a browser-like UA and unique identity per run. The webhook path already builds in a 5s test delay ([qstash.ts#L88](../../../apps/web/lib/webhook/qstash.ts#L88)).

## Recipe 6: Redirect behavior (middleware, no API)

Hit the short domain directly with `fetch(url, { redirect: "manual" })`; assert `status` 302 and the `location` header; `+`-suffix to assert inspect mode rewrites (200 + HTML). This is how to pin the guard branches (expired/password/banned) — see [ticket 6](../06-contribution-practice/01-good-first-tickets.md) for the full worked example.

## Checklist before you commit a test here

- [ ] Cleanup registered before the mutating call, tolerant of partial creation.
- [ ] No fixed `sleep` for async effects — poll with deadline.
- [ ] Unique names/keys via `randomId()` — the DB is shared and persistent.
- [ ] Asserting codes and shapes, not prose messages.
- [ ] Test tells a reader *which contract* it pins (name it after the behavior, not the endpoint).
