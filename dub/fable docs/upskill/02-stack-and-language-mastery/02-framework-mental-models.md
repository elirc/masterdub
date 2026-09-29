# Framework Mental Models (Next.js App Router + React + SWR)

## Next.js App Router: three request kinds

1. **Middleware** — runs before routing on (almost) every request; can rewrite (URL stays, content changes) or redirect. Dub's entire redirect product is middleware ([middleware.ts](../../../apps/web/middleware.ts)); the matcher at [#L21-L33](../../../apps/web/middleware.ts#L21-L33) is the *negative space* — `/api`, `_next`, metadata files bypass it.
2. **Server rendering (RSC)** — pages under `app/` are server components by default; route groups like `(dashboard)`, `(auth)` in [app/app.dub.co/](../../../apps/web/app/app.dub.co/) organize layouts without affecting URLs; `(ee)` is a license boundary expressed as a route group.
3. **Route handlers** — `route.ts` files exporting `GET`/`POST` ([app/api/links/route.ts](../../../apps/web/app/api/links/route.ts)). Dub wraps every one in an auth HOF.

Rewrites are the repo's signature move: the redirect engine *rewrites* to internal pages for not-found ([link.ts#L99-L104](../../../apps/web/lib/middleware/link.ts#L99-L104)), expired, banned, password, proxy, cloaking — the browser URL never changes, the served content does. Know rewrite-vs-redirect cold; it's both a framework question and a product behavior here.

**Mutations have two paths** in this repo, and choosing between them is a real design decision you should be able to defend:
- REST routes (public contract, used by dashboard *and* API customers) — [app/api/](../../../apps/web/app/api/)
- Server actions via next-safe-action (dashboard-only operations) — [lib/actions/safe-action.ts](../../../apps/web/lib/actions/safe-action.ts), with its own auth middleware chain (`actionClient` → `authUserActionClient` → `authActionClient`) that *re-implements* workspace membership. Two auth paths = two places to audit ([03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).

## React: what the repo's components teach

[ui/links/link-card.tsx](../../../apps/web/ui/links/link-card.tsx) is a compact masterclass:

- **`memo` at list-item boundaries** ([#L42](../../../apps/web/ui/links/link-card.tsx#L42), [#L51](../../../apps/web/ui/links/link-card.tsx#L51)) — list rerenders shouldn't rerender every card; memo only works if props are referentially stable, which is why the link object comes from SWR's cached array.
- **Context for subtree state** ([#L30-L40](../../../apps/web/ui/links/link-card.tsx#L30-L40)) — `showTests` is shared with descendants without prop-drilling; the `useLinkCardContext` throw-if-absent guard makes misuse loud.
- **Conditional data fetching** ([#L69-L72](../../../apps/web/ui/links/link-card.tsx#L69-L72)) — `useFolder({ enabled: showFolderIcon })`: don't fetch what you won't render. Hooks can't be called conditionally, but *fetching* can be.
- **Prefetch-on-visible** ([#L79-L84](../../../apps/web/ui/links/link-card.tsx#L79-L84)) — IntersectionObserver + `router.prefetch(editUrl)`: perceived-latency work. Note the `useEffect` dep array omits `editUrl` — predict when that's fine and when it's a stale-closure bug (drill below).
- **URL as state** — filters live in the querystring via `useRouterStuff`, so state is shareable/bookmarkable and survives refresh. The tradeoff: every filter change is a navigation.

## SWR: the data layer contract

Key = identity. [use-links.ts#L29-L50](../../../apps/web/lib/swr/use-links.ts#L29-L50) builds `/api/links?workspaceId=...&<filters>`; identical keys dedupe into one request and one cache row; `workspaceId` missing → key is null → fetch skipped. After mutations, [lib/swr/mutate.ts](../../../apps/web/lib/swr/mutate.ts) revalidates by key prefix. Stale-while-revalidate means users see cached data instantly and updates ripple in.

Failure modes to recognize anywhere: object-identity churn in options creating new keys each render; forgetting to mutate after writes (stale lists until refocus); fetching in effects instead of a keyed cache (no dedupe, race-prone).

## Pitfall checklist

- [ ] Is this component client-side for a reason (`"use client"` costs bundle)?
- [ ] Rewrite or redirect — did I pick for the URL semantics I actually want?
- [ ] Does my SWR key contain *every* input that changes the result?
- [ ] After this mutation, which keys are stale and who revalidates them?
- [ ] Is a server action or a REST route the right mutation surface — and is its auth equivalent?

## Drills

1. In `link-card.tsx`, the prefetch effect depends on `isInView` only. Construct the scenario where `editUrl` changes while mounted; is the stale closure harmful here? (Check what `editUrl` depends on and whether those change without remount.)
2. List every rewrite target in `LinkMiddleware` and what the browser URL shows for each.
3. Find one server action under [lib/actions/](../../../apps/web/lib/actions/) and its REST near-equivalent; compare their authorization line by line.

## Interview angle

- Rendering model / RSC vs client → [08/02 Q1–Q3](../08-interview-prep/02-frontend-framework-questions.md)
- memo/context/stale closures → [08/02 Q4–Q7](../08-interview-prep/02-frontend-framework-questions.md)
- SWR/React Query caching → [08/02 Q8](../08-interview-prep/02-frontend-framework-questions.md) and key-flows Flow 7
- rewrite vs redirect → comes up in system design; see [08/04 step 3](../08-interview-prep/04-system-design-from-this-repo.md)
