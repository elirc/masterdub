# Frontend Framework Question Cards (React / Next.js)

Ten cards. Same protocol: answer aloud for 90 seconds before reading.

---

## Q1: Server components vs client components — how do you decide?

Round: frontend
What it's really testing: whether you understand *where code runs* rather than memorizing "use client".
Repo anchor: pages under [app/app.dub.co/(dashboard)](../../../apps/web/app/app.dub.co/(dashboard)) are RSC by default; interactive pieces like [link-card.tsx](../../../apps/web/ui/links/link-card.tsx) are client (hooks everywhere).
Junior answer sounds like: "use client when you need useState."
Mid-level answer adds: the real dividing lines — interactivity/browser APIs/stateful hooks → client; data-close rendering, secrets, heavy deps → server; client components ship bytes to every visitor, so the boundary is a *bundle budget* decision too.
Senior answer includes: the boundary is transitive (a client component's imports all become client); design components so the interactive leaf is small and the static shell stays server; this repo predates-and-mixes patterns — dashboards are SWR-heavy client trees, a deliberate choice for a highly interactive app where personalization defeats static optimization anyway.
Likely follow-ups: "Where do server actions fit?" ([safe-action.ts](../../../apps/web/lib/actions/safe-action.ts) — mutations without API routes, but note the dual-auth-surface cost).
Practice drill: pick one dashboard page and classify each component in its tree server/client, with the byte argument.

## Q2: What actually happens on a React re-render, and when is one "wasted"?

Round: frontend
What it's really testing: render vs commit distinction.
Repo anchor: [link-card.tsx#L42-L51](../../../apps/web/ui/links/link-card.tsx#L42-L51) — `memo` on both card layers.
Junior answer sounds like: "the component runs again, which is slow."
Mid-level answer adds: render = calling the function and diffing; commit = DOM writes; renders are cheap until trees are big; `memo` skips re-rendering children whose props are referentially equal — which is why a 100-card list wraps each card in memo: list-level state changes (selection, filters) shouldn't re-run 100 card functions.
Senior answer includes: memo is defeated by fresh object/function props (referential identity!), which is the *actual* bug to look for; measure with the Profiler before memoizing (see [kata 6 in review katas](../04-code-reading-gym/04-review-katas.md) — the useMemo-everywhere PR you should decline).
Likely follow-ups: "when is useMemo itself a cost?"
Practice drill: find what props `LinkCard` receives and argue whether its memo can ever actually skip work given where `link` objects come from (SWR cached array — stable until revalidation).

## Q3: Stale closures in hooks — give a real example.

Round: frontend
What it's really testing: effect dependency reasoning beyond lint-rule obedience.
Repo anchor: [link-card.tsx#L82-L84](../../../apps/web/ui/links/link-card.tsx#L82-L84) — the prefetch effect depends on `[isInView]` but uses `editUrl`.
Junior answer sounds like: recites the setInterval counter example.
Mid-level answer adds: this real case — if `editUrl` changed between renders while `isInView` stayed true, the effect wouldn't re-run and could prefetch a stale URL; then evaluates *severity*: `editUrl` derives from slug/domain/key which don't change without a remount in this list, and prefetch is best-effort — so the omission is harmless-by-context, though exhaustive-deps would flag it.
Senior answer includes: the general rule — dependency arrays are about *values the closure reads*, and "it can't change in practice" is a bet that should be written as a comment or made structural; knowing when to accept the lint suppression is judgment, not rebellion.
Likely follow-ups: "why does the linter want editUrl there? would adding it break anything?" (No — it's memoized; adding it is free. That's the actual right move.)
Practice drill: explain aloud why adding `editUrl` to the deps is strictly better and costs nothing here.

## Q4: How do you share state between components — walk through your decision ladder.

Round: frontend
What it's really testing: state placement judgment.
Repo anchor: three tiers in one feature — local `useState` for `showTests`, lifted into `LinkCardContext` for the card subtree ([link-card.tsx#L30-L48](../../../apps/web/ui/links/link-card.tsx#L30-L48)); URL querystring for filters (via `useRouterStuff`); SWR cache as *server-state* store ([use-links.ts](../../../apps/web/lib/swr/use-links.ts)).
Junior answer sounds like: "useState, or Redux for global state."
Mid-level answer adds: the ladder — local state → lift → context (subtree, low-frequency) → URL (shareable/bookmarkable UI state) → server-state cache (SWR/React Query — *not* your state, the server's) → only then a store; and identifies filters-in-URL as the underrated tier (survives refresh, deep-linkable, back-button-correct).
Senior answer includes: server state and client state have different consistency needs — copying SWR data into useState is the classic bug (two sources of truth); context re-renders every consumer on value change, hence keeping `LinkCardContext` values small and subtree-scoped; the throw-if-no-provider guard ([#L35-L40](../../../apps/web/ui/links/link-card.tsx#L35-L40)) as API design for hooks.
Likely follow-ups: "why is putting fetched data in Redux usually wrong?"
Practice drill: for five pieces of state in the links page (filters, selection, modal open, links data, workspace), name the correct tier and find where the repo put each.

## Q5: How does SWR (or React Query) actually work?

Round: frontend
What it's really testing: the cache model, not the API.
Repo anchor: [use-links.ts#L11-L50](../../../apps/web/lib/swr/use-links.ts#L11-L50); mutation helpers [lib/swr/mutate.ts](../../../apps/web/lib/swr/mutate.ts).
Junior answer sounds like: "it fetches and caches."
Mid-level answer adds: key = identity; stale-while-revalidate (serve cache, refetch in background, swap); dedup of same-key requests across components; conditional fetch via null keys (`workspaceId` not loaded yet); after mutations you invalidate keys — in this repo by prefix, because filter permutations make exact keys unknowable.
Senior answer includes: the cache is *per-key-string*, so unstable option objects fragment it; refocus/reconnect revalidation as the freshness backstop; and when SWR is the wrong tool (truly real-time data, or server-rendered pages that shouldn't double-fetch).
Likely follow-ups: "optimistic updates — how, and what's the rollback story?"
Practice drill: archive a link mentally: list every SWR key now stale (list, counts, the link itself) and find the repo helper that revalidates them.

## Q6: The links dashboard feels slow with 100 links. Diagnose.

Round: frontend / practical
What it's really testing: a measurement-first mindset applied to UI.
Repo anchor: [links-container.tsx](../../../apps/web/ui/links/links-container.tsx), [link-card.tsx](../../../apps/web/ui/links/link-card.tsx).
Junior answer sounds like: "add React.memo and useMemo."
Mid-level answer adds: profile first — is it render time (Profiler), network waterfalls (each card's conditional `useFolder` — [#L69-L72](../../../apps/web/ui/links/link-card.tsx#L69-L72) — how many distinct folder fetches?), or main-thread work (favicon/image decoding)? Then: the repo already memos cards and gates folder fetches by `enabled` — so measure before assuming the obvious fixes are missing.
Senior answer includes: the API-shape option (include folder data server-side — with the ACL caveat from [kata 7](../06-contribution-practice/04-refactor-and-design-katas.md)), virtualization thresholds, and the observation that the *product* already adapts to scale (MEGA workspaces get exact search / no archived — [use-links.ts#L36-L40](../../../apps/web/lib/swr/use-links.ts#L36-L40)) — degrading features is a legitimate perf tool.
Practice drill: write the 4-step measurement plan you'd run before changing any code.

## Q7: How do you handle loading, error, and empty states without making a mess?

Round: frontend
What it's really testing: production polish.
Repo anchor: `link-card-placeholder.tsx` and `link-not-found.tsx` in [ui/links/](../../../apps/web/ui/links/); the `CardList.Context` `loading` flag consumed inside cards ([link-card.tsx#L52](../../../apps/web/ui/links/link-card.tsx#L52)).
Junior answer sounds like: "if (isLoading) return spinner."
Mid-level answer adds: the trio is a state machine per data source (loading / error / empty / data — four, not three); skeletons that match layout (placeholder components) beat spinners for perceived speed; SWR's `isValidating` vs no-data distinction (background refresh shouldn't blank the screen).
Senior answer includes: loading states propagated via context so leaf components render skeleton variants without prop-drilling (the `loading` flag pattern here); error boundaries for render errors vs data errors; empty states as *product* surfaces (first-run experience), not afterthoughts.
Practice drill: trace how `loading` reaches `LinkCardInner` and what renders differently.

## Q8: Accessibility — what do you actually check?

Round: frontend
What it's really testing: whether a11y is in your definition of done.
Repo anchor: conceptual — audit target: [ui/links/link-controls.tsx](../../../apps/web/ui/links/link-controls.tsx) (menus/buttons); the repo uses Radix primitives (`@radix-ui/*` in [package.json](../../../apps/web/package.json)) which carry a11y defaults.
Junior answer sounds like: "alt tags and aria labels."
Mid-level answer adds: keyboard-first audit (tab order, escape closes modals, focus trap + restore), semantic elements before ARIA, contrast, and *leaning on primitives* (Radix/headless libs) instead of hand-rolling dropdowns — the highest-leverage a11y decision in modern React.
Senior answer includes: a11y as architecture — clickable cards (this repo's `useClickHandlers` on a div-based card, [link-card.tsx#L92](../../../apps/web/ui/links/link-card.tsx#L92)) need a real link/button inside for keyboard/AT users — worth auditing here; automated checks (axe) catch ~30%, the rest is keyboard testing.
Practice drill: keyboard-walk the links list in a running instance (or reason from code): can you open link controls without a mouse?

## Q9: What's your bundle-size discipline?

Round: frontend
What it's really testing: whether you think about what ships.
Repo anchor: heavyweight deps in [apps/web/package.json](../../../apps/web/package.json) (tiptap editor, react-pdf, recharts-class libs) — candidates for dynamic import; shared `@dub/ui` package boundary.
Junior answer sounds like: "code splitting."
Mid-level answer adds: measure (`next build` output, analyzer), route-level splitting is automatic in Next — the discipline is *not importing heavy things into common layouts*; `dynamic(() => import(...))` for editors/charts/PDF that render behind interaction; watch barrel files re-exporting the world.
Senior answer includes: the client/server boundary as the *primary* bundle tool in App Router (move it server-side and it ships nothing); per-PR bundle budgets in CI; the monorepo wrinkle — a "utils" package import can drag surprising weight into every consumer.
Practice drill: find where tiptap is imported and determine (from code) whether it's dynamically loaded or in a shared path.

## Q10: Forms and validation — client, server, or both?

Round: frontend
What it's really testing: trust boundaries applied to UX.
Repo anchor: link builder ([ui/links/link-builder/](../../../apps/web/ui/links/link-builder/)) + the same Zod schemas enforced server-side ([links route L54-L56](../../../apps/web/app/api/links/route.ts#L54-L56)).
Junior answer sounds like: "validate on the client for UX."
Mid-level answer adds: both, same rules — client for immediacy, server as the actual boundary (client validation is a UX feature, not security); sharing the Zod schema between form resolver and API kills drift; server errors must still render field-level (map the 422 error paths back to inputs).
Senior answer includes: async validation (key availability) needs debounce + race handling (out-of-order responses); and the deeper contract point — the *server's* error taxonomy ([errors.ts](../../../apps/web/lib/api/errors.ts)) is what makes field-mapping possible; prose-only errors doom the client to toast-blindness.
Practice drill: find how the link builder checks key availability (`/api/links/exists` — [app/api/links/exists](../../../apps/web/app/api/links/exists)) and whether rapid typing can race.
