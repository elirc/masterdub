# Boundaries and Layers

A **boundary** is a line where responsibility changes hands and assumptions must be re-established. A layer *owns* certain decisions; a leak is when a decision escapes its layer.

## The layer cake (API side)

```
route handler (app/api/**/route.ts)      owns: HTTP — status codes, headers, wiring
  └─ auth wrapper (lib/auth/*)           owns: identity, tenancy, permissions, rate, plan
      └─ contract (lib/zod/schemas/*)    owns: request/response shape
          └─ domain (lib/api/<resource>) owns: business rules, invariants, orchestration
              └─ stores (prisma | planetscale | upstash | tinybird)  own: durability, atomicity
```

What each layer must NOT own: routes must not contain business rules (compare how thin [links route](../../../apps/web/app/api/links/route.ts) is); domain functions must not read HTTP headers (they take parsed args); stores must not enforce tenancy on their own (they receive `workspaceId` — the domain provides it).

## Good boundaries, with evidence

- **HTTP ↔ domain**: `processLink` never sees a `Request`; it takes a payload and returns a value-union ([process-link.ts#L24-L57](../../../apps/web/lib/api/links/process-link.ts#L24-L57)). Result: the same function serves single, bulk, upsert, and update endpoints.
- **Policy ↔ handler**: `requiredPermissions: ["links.write"]` lives *next to the handler that needs it* ([links route L107-L109](../../../apps/web/app/api/links/route.ts#L107-L109)) while enforcement lives in the wrapper — declaration and mechanism separated.
- **Contract ↔ everything**: schemas in one directory, feeding validation, types, and OpenAPI. The webhook payload being parsed before *sending* ([links route L92](../../../apps/web/app/api/links/route.ts#L92)) shows contract-consciousness on the way out, not just in.
- **App ↔ analytics store**: JS never assembles analytics SQL; Tinybird pipes ([packages/tinybird](../../../packages/tinybird)) own aggregation. The app sends parameters and gets rows.

## Boundary leaks and tensions, with evidence

- **Wire format leaking inward**: `clickData` is snake_case throughout [record-click.ts#L134-L169](../../../apps/web/lib/tinybird/record-click.ts#L134-L169) because Tinybird columns are — the store's naming convention shapes app-internal code. Cheap leak, consciously accepted; know that it *is* one.
- **Cache shape as unspoken contract**: the redirect destructures 16 fields from `cachedLink` ([link.ts#L134-L151](../../../apps/web/lib/middleware/link.ts#L134-L151)) that `formatRedisLink` must have included — enforced by `as any`, i.e., by nothing. This is the repo's clearest example of an **implicit invariant** (a rule the code depends on but nothing enforces).
- **Two mutation surfaces, two policies**: REST wrappers ([workspace.ts](../../../apps/web/lib/auth/workspace.ts)) vs server-action middleware ([safe-action.ts](../../../apps/web/lib/actions/safe-action.ts)) implement overlapping auth separately. Duplicated policy drifts; when auditing, you must read both.
- **The god-wrapper**: `withWorkspace` owns identity + rate + tenancy + roles + scopes + plans + flags + logging — eight concerns in one function. The alternative (composed middlewares) has its own costs (ordering bugs); this is a *tension*, not a verdict.
- **Cross-file compensating controls**: anonymous link creation's protections live in three files (carve-out in the wrapper, rate limit in the route, self-delete in create-link) — the security boundary is smeared. See [ticket 5](../06-contribution-practice/01-good-first-tickets.md).

## The transferable test

For any file, ask: *what decisions does this file own, and if I changed one, who else would have to know?* If the answer is "a file in a different layer, silently," you've found a leak. Practice on [update-link.ts](../../../apps/web/lib/api/links/update-link.ts): does it own cache invalidation, or does the cache own knowing about updates?

Interview angle: "how do you structure a Node/Next API?" — describe this cake with the two "must NOT own" rules; that's a mid-level answer. Adding one leak you've seen and its cost is the senior answer. Cross-link: [08/03 Q1](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: pick [app/api/tags](../../../apps/web/app/api/tags) (unexamined in this curriculum). Map its route → wrapper → schema → domain files, then find one boundary respected and one bent. Time-box: 25 minutes.
