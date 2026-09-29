# Type System and Contracts (TypeScript + Zod)

## Mental model

TypeScript types exist only at compile time; every byte crossing a process boundary (HTTP body, Redis value, queue message, DB row through raw SQL) arrives untyped at runtime. A mature codebase therefore has **two type systems**: static (TS) and runtime (Zod), joined by `z.infer` so they cannot drift. Where the repo has only the static half (raw SQL rows, Redis JSON), you'll find `as` casts — each one a small hole in the boundary.

## Where the repo uses it well

- **Parse at the boundary, trust inside**: [links route L54-L56](../../../apps/web/app/api/links/route.ts#L54-L56) parses the body; everything downstream takes typed `NewLinkProps`. 70+ schemas in [lib/zod/schemas/](../../../apps/web/lib/zod/schemas/) are the API's real specification — OpenAPI is generated from them ([scripts/generate-openapi.ts](../../../apps/web/scripts/generate-openapi.ts)).
- **Outbound parsing**: webhook payloads are parsed *before sending* ([links route L92](../../../apps/web/app/api/links/route.ts#L92)) — producing to a public contract is validated like consuming.
- **Discriminated unions for fallible operations**: [process-link.ts#L44-L57](../../../apps/web/lib/api/links/process-link.ts#L44-L57), with `code?: never` on the success arm so you can't read an error code off a success.
- **Literal-derived unions**: `PERMISSION_ACTIONS as const` → `PermissionAction` ([permissions.ts#L3-L31](../../../apps/web/lib/api/rbac/permissions.ts#L3-L31)) — the string list *is* the type; adding a permission updates both.
- **Generics that thread caller context**: `processLink<T>` preserving extra payload fields ([#L24-L34](../../../apps/web/lib/api/links/process-link.ts#L24-L34)).
- **Schema-per-purpose**: create vs update vs query vs response schemas in [links.ts](../../../apps/web/lib/zod/schemas/links.ts) — resist "one schema to rule them all"; a PATCH contract legitimately differs from a POST contract.

## Where it has sharp edges

- `as any` at the Redis/middleware seam: [link.ts#L110](../../../apps/web/lib/middleware/link.ts#L110), [#L115](../../../apps/web/lib/middleware/link.ts#L115) — the cached shape is enforced by convention (`formatRedisLink`), not by types. Cache-shape drift is invisible to the compiler (see ticket 13).
- Raw SQL rows: `res.rows[0] as any` ([record-click.ts#L344](../../../apps/web/lib/tinybird/record-click.ts#L344)) — schema changes break this at runtime only.
- `@ts-expect-error` at the anonymous-auth carve-out ([workspace.ts#L140](../../../apps/web/lib/auth/workspace.ts#L140)) — the type system correctly rejects the call; the suppression records a deliberate contract violation. Good use of `expect-error` over `ignore` (it errors when the underlying issue is fixed); still a debt marker.
- `Record<string, any>` generic bounds — permissive by design, but `any` propagates.

## unknown vs any, narrowing, satisfies — the 90-second versions

- `any` disables checking and **spreads** through everything it touches; `unknown` is the safe top type — you must narrow before use. Boundary code should return `unknown` (or parsed types), never `any`.
- Narrowing: `if (error != null)` on the processLink union; `instanceof` chains in [handleApiError](../../../apps/web/lib/api/errors.ts#L92-L130) (Zod → DubApiError → Prisma-code → fallthrough) — error handling *is* type narrowing at runtime.
- `satisfies` checks a value against a type **without widening it** — useful for config objects where you want both checking and precise inference. (No load-bearing instance found in the files read; treat as conceptual here.)

## Pitfall checklist

- [ ] Does data crossing a process boundary get parsed, or just asserted (`as`)?
- [ ] Is my error path narrowable (discriminant field), or `catch (e: any)`?
- [ ] Did I create a second source of truth for a shape that Zod already defines (`z.infer` instead)?
- [ ] Any new `as any` — can a schema or generic remove it?

## Drills

1. Count the `as any` / `@ts-ignore` / `@ts-expect-error` instances in [link.ts](../../../apps/web/lib/middleware/link.ts), [record-click.ts](../../../apps/web/lib/tinybird/record-click.ts), [workspace.ts](../../../apps/web/lib/auth/workspace.ts); classify each as boundary-hole vs deliberate-violation vs laziness.
2. Sketch the Zod schema that would type the raw SQL webhook row at [record-click.ts#L324-L348](../../../apps/web/lib/tinybird/record-click.ts#L324-L348) and where you'd parse it.
3. From [permissions.ts](../../../apps/web/lib/api/rbac/permissions.ts), explain what breaks at compile time when you add `"campaigns.write"` to the const array but not to any role.

## Interview angle

- runtime vs compile-time validation → [08/01 Q4](../08-interview-prep/01-js-ts-node-deep-dive.md)
- discriminated unions / result types → [08/01 Q3](../08-interview-prep/01-js-ts-node-deep-dive.md)
- generics that earn their keep → [08/01 Q8](../08-interview-prep/01-js-ts-node-deep-dive.md)
- API contract evolution → [08/03 Q6, Q10](../08-interview-prep/03-api-and-data-modeling-questions.md)
