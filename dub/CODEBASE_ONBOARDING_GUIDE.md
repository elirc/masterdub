# Codebase Onboarding Guide

## 1. Executive Summary

Dub is an open-source link attribution platform for short links, conversion tracking, analytics, and affiliate/partner programs. The repository is a TypeScript monorepo: the main product is a Next.js application in `apps/web`, supported by shared packages for Prisma, UI, utilities, email, embeds, integrations, and a CLI.

The core mental model is: Dub receives traffic on many hostnames, routes it through middleware, resolves short links from cache/database, records events asynchronously, and exposes a dashboard/API for managing links, workspaces, domains, partners, webhooks, analytics, and billing.

For a new developer, think of the system as five cooperating layers:

1. Host-aware routing and redirects in `apps/web/middleware.ts`.
2. App Router UI and API routes in `apps/web/app`.
3. Domain/business services in `apps/web/lib`.
4. Shared packages in `packages/*`.
5. External systems: PlanetScale/MySQL via Prisma, Upstash Redis/QStash, Tinybird, NextAuth, Stripe, Resend, Vercel, storage, and integration APIs.

## 2. How To Run The Project

Package manager: `pnpm@9.15.9`, declared in `package.json`.

Recommended versions from `README.md`:

- Node: `v23.11.0`
- pnpm: `9.15.9`

Install from the repository root:

```bash
pnpm install
```

Common root commands:

```bash
pnpm dev
pnpm build
pnpm lint
pnpm test
pnpm prettier-check
```

Main web app commands:

```bash
pnpm --filter web dev
pnpm --filter web build
pnpm --filter web test
pnpm --filter web test:e2e
pnpm --filter web prisma:generate
pnpm --filter web prisma:push
pnpm --filter web prisma:studio
```

`apps/web/package.json` runs the local app on port `8888`. The `dev` script runs Prisma generation, `next dev --turbopack --port 8888`, and Prisma Studio.

Important setup files:

- `apps/web/.env.example`: required and optional environment variables.
- `apps/web/docker-compose.yml`: local MySQL, PlanetScale HTTP simulator, and MailHog.
- `packages/prisma/schema/*.prisma`: split Prisma schema files.
- `turbo.json`: monorepo pipeline.
- `pnpm-workspace.yaml`: workspace packages.

Required local services for realistic development:

- MySQL on `3306`.
- PlanetScale HTTP simulator on `3900`.
- MailHog SMTP/API on `1025`/`8025`.
- Upstash Redis/QStash, Tinybird, Resend, Vercel Domains API, and secrets from `apps/web/.env.example` for full behavior.

A successful local setup usually means:

- `pnpm install` completes.
- `cp apps/web/.env.example apps/web/.env` has been done and required values are filled.
- Local database services are running.
- `pnpm --filter web prisma:generate` works.
- `pnpm --filter web prisma:push` creates tables.
- `pnpm --filter web dev` starts Next.js at `http://localhost:8888` and Prisma Studio.

If Prisma errors say a table does not exist, run `pnpm --filter web prisma:push`. For seed data, the README documents:

```bash
cd apps/web
pnpm run script dev/seed
```

## 3. Repository Map

`apps/web`
: The main Next.js 15 / React 19 application. It contains the dashboard, public short-link routes, API routes, cron endpoints, middleware, domain logic, integration logic, tests, scripts, and local service config.

`apps/web/app`
: Next.js App Router tree. This includes layouts, pages, API route handlers, cron route handlers, host-specific app folders like `app.dub.co`, public domain routes like `[domain]`, and route groups like `(ee)`.

`apps/web/middleware.ts`
: The traffic director. It parses the request host/path, then dispatches to app, API, admin, partners, short-link, stats, `.well-known`, or link-creation middleware.

`apps/web/lib`
: Server and shared application logic. This is where most business behavior lives: `api`, `auth`, `middleware`, `tinybird`, `webhook`, `zod`, `swr`, integrations, cron/QStash, storage, billing, analytics, and domain services.

`apps/web/lib/api`
: Server-side business functions called by route handlers. Route files should stay thin; this folder owns the real work.

`apps/web/lib/zod/schemas`
: Request/response validation schemas, often also annotated for OpenAPI generation.

`apps/web/lib/swr`
: Client-side data hooks used by interactive UI components.

`apps/web/ui`
: App-specific React UI grouped by product domain: links, analytics, workspaces, partners, folders, auth, modals, layout, integrations, and more.

`apps/web/tests`
: Vitest API/domain tests, organized by feature area.

`apps/web/playwright`
: Playwright end-to-end tests for partner and workspace flows.

`apps/web/scripts`
: Operational scripts, seed scripts, backfills, importers, and OpenAPI generation.

`packages/prisma`
: Prisma client package. `index.ts` exports the main singleton client, `edge.ts` exports a PlanetScale adapter client, and `schema/*.prisma` holds the database model split by domain.

`packages/utils`
: Shared constants and pure utility functions used across the app and packages. Examples include domain helpers, URL helpers, formatting, IDs, and pricing constants.

`packages/ui`
: Shared UI primitives and components, built with `tsup`, React, Radix, Tailwind, Floating UI, Visx, and related libraries.

`packages/email`
: Email templates and email helpers used by the app.

`packages/embeds/core` and `packages/embeds/react`
: Embeddable client-side packages for Dub features.

`packages/cli`
: Published `dub` CLI for shortening/managing URLs through the Dub API.

`packages/tinybird`
: Tinybird datasources and pipes for analytics/event storage.

`packages/stripe-app` and `packages/hubspot-app`
: Integration-specific packages.

`packages/tsconfig`
: Shared TypeScript configs. Packages tend to be stricter than `apps/web`.

## 4. System Architecture

Dub is a host-aware Next.js monolith inside a Turborepo. The main app uses the App Router for UI/API routes and middleware for high-volume short-link traffic. Shared packages are publishable or reusable libraries.

```mermaid
flowchart TD
  Browser[Browser or API Client] --> MW[apps/web/middleware.ts]
  MW --> AppHost[App dashboard routes<br/>apps/web/app/app.dub.co]
  MW --> ApiHost[API routes<br/>apps/web/app/api]
  MW --> LinkFlow[Short-link flow<br/>apps/web/lib/middleware/link.ts]
  MW --> AdminPartners[Admin and partners middleware]

  AppHost --> UI[apps/web/ui]
  UI --> SWR[apps/web/lib/swr]
  SWR --> ApiHost

  ApiHost --> Auth[withWorkspace / withSession / withAdmin]
  Auth --> Zod[lib/zod/schemas]
  Zod --> Services[lib/api domain services]
  Services --> Prisma[@dub/prisma]
  Services --> Redis[Upstash Redis]
  Services --> TB[Tinybird]
  Services --> QStash[Upstash QStash]
  Services --> External[Stripe, Resend, Vercel, Slack, Shopify, etc.]

  LinkFlow --> LinkCache[Redis link cache]
  LinkFlow --> EdgeDB[PlanetScale edge reads]
  LinkFlow --> Redirect[Redirect/rewrite response]
  LinkFlow --> AsyncEvents[ev.waitUntil click/event work]
  AsyncEvents --> TB
  AsyncEvents --> Redis
  AsyncEvents --> QStash
```

Main runtime flow for dashboard/API requests:

1. Request enters Next.js.
2. `apps/web/middleware.ts` routes by hostname and path.
3. App pages under `apps/web/app/app.dub.co` render server components and client components.
4. Client components call SWR hooks in `apps/web/lib/swr`.
5. API route handlers under `apps/web/app/api` authenticate with wrappers like `withWorkspace`.
6. Route handlers validate input with Zod schemas from `apps/web/lib/zod/schemas`.
7. Domain services in `apps/web/lib/api/*` execute business logic.
8. Data is read/written through `@dub/prisma`, Tinybird, Redis, QStash, or third-party APIs.
9. API responses are normalized through `NextResponse.json`, and errors use `DubApiError`.

Main runtime flow for a short link:

1. A request to a custom domain or Dub short domain enters `apps/web/middleware.ts`.
2. The middleware parses domain/key via `apps/web/lib/middleware/utils/parse`.
3. `LinkMiddleware` in `apps/web/lib/middleware/link.ts` normalizes and resolves the key.
4. It reads link data from Redis cache through `linkCache`; on miss it reads from PlanetScale via `getLinkViaEdge`.
5. It handles inspect mode, password protection, banned/disabled/expired links, proxy/cloak pages, custom URI schemes, A/B variants, device targeting, and geo targeting.
6. It returns a redirect or rewrite.
7. It records click/event side effects asynchronously with `ev.waitUntil`.

Where side effects happen:

- Database writes: `apps/web/lib/api/**`, auth callbacks, cron handlers, webhook handlers.
- Click/event analytics: `apps/web/lib/tinybird/**` and `packages/tinybird`.
- Caching/rate limiting: `apps/web/lib/upstash/**`.
- Queues/background jobs: `apps/web/lib/cron`, QStash calls in services like link deletion, webhook delivery, campaigns, imports.
- External webhooks: route handlers in `apps/web/app/api/**` and `apps/web/app/(ee)/api/**`.
- File/image storage: `apps/web/lib/storage`.
- Email: `@dub/email` and Resend integration.

State management:

- Server state is persisted in MySQL/PlanetScale through Prisma.
- Analytics/event state is stored and queried in Tinybird.
- Hot-path cache and locks live in Upstash Redis.
- Client state is mostly local React state plus SWR cache.
- Auth state is NextAuth JWT session state with a Prisma adapter.

## 5. Important Components And How They Interact

`apps/web/middleware.ts`

- Responsibility: host-aware dispatch and short-link routing.
- Inputs: `NextRequest`, `NextFetchEvent`.
- Outputs: `NextResponse` redirect, rewrite, or pass-through.
- Calls: `AppMiddleware`, `ApiMiddleware`, `AdminMiddleware`, `PartnersMiddleware`, `CreateLinkMiddleware`, `LinkMiddleware`.
- Depends on: hostname constants from `@dub/utils`, parser helpers, Axiom logging.
- Risk: changing this can break all host routing, public redirects, stats pages, API host behavior, and app dashboard access.

`apps/web/lib/middleware/link.ts`

- Responsibility: resolve short links and decide redirect/rewrite behavior.
- Inputs: request domain/key, cookies, user agent, geo data.
- Outputs: redirect/rewrite responses and click ID cookies.
- Calls: `linkCache`, `getLinkViaEdge`, `recordClick`, `getPartnerEnrollmentInfo`, deep-link helpers.
- Depends on: Redis, PlanetScale, Tinybird, Vercel request APIs, utility constants.
- Risk: this is a high-traffic path; bugs affect redirects, click attribution, bot/proxy behavior, and conversion tracking.

`apps/web/app/api/links/route.ts`

- Responsibility: public/workspace link list and create API.
- Inputs: query params or request body.
- Outputs: JSON link responses.
- Calls: `withWorkspace`, `validateLinksQueryFilters`, `getLinksForWorkspace`, `processLink`, `createLink`, `sendWorkspaceWebhook`.
- Depends on: Zod link schemas, auth wrapper, rate limit, domain services.
- Risk: API compatibility, permissions, usage limits, webhook behavior, and link creation semantics.

`apps/web/lib/api/links/create-link.ts`

- Responsibility: persist a processed link and trigger follow-up work.
- Inputs: `ProcessedLinkProps`.
- Outputs: transformed link response.
- Calls: Prisma create, image upload, `linkCache.set`, `recordLink`, QStash deletion for anonymous links, usage stream publishing, webhook propagation, A/B test scheduler.
- Depends on: Prisma, Redis, Tinybird, storage, QStash, partner enrollment lookup.
- Risk: changing it can desync database, cache, analytics, usage billing, webhooks, and image behavior.

`apps/web/lib/auth/workspace.ts`

- Responsibility: API wrapper for workspace-aware routes.
- Inputs: route handler and policy options.
- Outputs: wrapped handler with auth, permission checks, workspace context, logging, and error normalization.
- Calls: session/token lookup, workspace resolution, RBAC/plan checks, request logging.
- Depends on: NextAuth, Prisma, tokens, API logs, rate limits.
- Risk: affects most workspace API routes and the public API surface.

`apps/web/lib/auth/options.ts`

- Responsibility: NextAuth provider and session configuration.
- Inputs: OAuth, email, SAML, credentials callbacks.
- Outputs: session/JWT behavior and sign-in side effects.
- Calls: Prisma adapter, email sending, SAML Jackson, avatar storage, QStash welcome workflow, program application completion.
- Depends on: NextAuth, Prisma, Resend/email templates, BoxyHQ, storage, QStash.
- Risk: can break login, SSO enforcement, user creation, session cookies, and onboarding side effects.

`packages/prisma/index.ts` and `packages/prisma/edge.ts`

- Responsibility: database clients.
- Inputs: `DATABASE_URL` / `PLANETSCALE_DATABASE_URL`.
- Outputs: Prisma clients.
- Calls: Prisma Client and PlanetScale adapter.
- Depends on: generated Prisma schema.
- Risk: affects all persistence. The main client omits `user.passwordHash` by default.

`apps/web/lib/zod/schemas/*`

- Responsibility: validation, API contract typing, and OpenAPI metadata.
- Inputs: raw request bodies, query params, external webhook payloads.
- Outputs: parsed typed values or Zod errors.
- Calls: Zod v4.
- Depends on: app domain types and constants.
- Risk: schema changes are API changes; OpenAPI output can drift if metadata is missed.

`apps/web/lib/openapi/index.ts`

- Responsibility: assemble API docs from schemas and path definitions.
- Inputs: per-domain OpenAPI definitions under `apps/web/lib/openapi`.
- Outputs: OpenAPI document.
- Depends on: Zod schemas.
- Risk: client SDKs/docs can become inaccurate if route behavior and schemas diverge.

`apps/web/lib/webhook/publish.ts` and `apps/web/lib/webhook/qstash.ts`

- Responsibility: publish workspace webhooks and enqueue delivery.
- Inputs: trigger, workspace, event payload.
- Outputs: QStash jobs and delivery logs/callbacks.
- Depends on: Redis/QStash, Prisma, webhook schemas.
- Risk: customer integrations depend on event shape and delivery reliability.

`apps/web/lib/tinybird/*`

- Responsibility: record and query click, lead, sale, link, API log, and analytics data.
- Inputs: events from middleware/API/integrations.
- Outputs: Tinybird writes and query results.
- Depends on: Tinybird API key/URL and schemas.
- Risk: analytics, attribution, and billing-adjacent reporting depend on this layer.

`apps/web/lib/actions/safe-action.ts`

- Responsibility: typed server-action clients for authenticated UI mutations.
- Inputs: action payloads and session/workspace/partner context.
- Outputs: safe action results consumed by `next-safe-action/hooks`.
- Depends on: auth helpers and Zod schemas.
- Risk: UI mutation auth and validation behavior changes here.

## 6. TypeScript Design

The repository uses TypeScript throughout, but strictness varies by layer.

- Shared packages generally extend strict configs from `packages/tsconfig/base.json`.
- The Next app extends `packages/tsconfig/nextjs.json`, then sets `strict: false` and `strictNullChecks: true` in `apps/web/tsconfig.json`.
- Web aliases are local: `@/lib/*`, `@/ui/*`, `@/pages/*`, `@/styles/*`.
- Shared packages are imported by package name: `@dub/prisma`, `@dub/ui`, `@dub/utils`, `@dub/email`.

Important type locations:

- `apps/web/lib/types.ts`: application/domain-facing types.
- `apps/web/lib/zod/schemas/*.ts`: inferred request/response types.
- `packages/prisma/client.ts`: Prisma generated type re-export.
- `packages/prisma/schema/*.prisma`: source of database model types.
- `packages/ui/src/table/types.ts`: example of component-level shared types.

Validation/parsing strategy:

- The app uses `zod/v4` for most web schemas.
- Route handlers parse query/body data before calling domain services.
- Example from `apps/web/app/api/links/route.ts`: `getLinksQuerySchemaExtended.parse(searchParams)` and `createLinkBodySchemaAsync.parseAsync(await parseRequestBody(req))`.
- `parseRequestBody` in `apps/web/lib/api/utils.ts` normalizes invalid JSON and throws `DubApiError`.
- Schemas often carry `.meta()` and descriptions for OpenAPI in `apps/web/lib/openapi`.

Error handling:

- API code throws `DubApiError` from `apps/web/lib/api/errors.ts`.
- Error codes live in `apps/web/lib/api/error-codes.ts`.
- Zod errors become `422 unprocessable_entity`.
- Prisma `P2025` becomes `404 not_found`.
- Unknown errors become a generic `500 internal_server_error`.
- SWR/client fetch errors are normalized by `packages/utils/src/functions/fetcher.ts`.

Advanced TypeScript patterns:

- Zod schema inference for API contracts.
- Prisma types and `Pick<PrismaClient[...]...>` in testable service functions.
- Route wrappers that provide typed context objects, especially `withWorkspace`.
- Shared package export maps in `packages/ui/package.json` and `packages/prisma/package.json`.
- Some pragmatic escape hatches exist: `@ts-ignore`, `@ts-expect-error`, `any`, and `strict: false` in the web app.

Naming conventions:

- Route handlers use HTTP verb exports: `GET`, `POST`, `PATCH`, `DELETE`.
- Domain services use verb-object names: `createLink`, `getLinksForWorkspace`, `validateLinksQueryFilters`.
- Zod schemas use object-domain names: `createLinkBodySchemaAsync`, `linkEventSchema`, `getLinksQuerySchemaExtended`.
- Hooks use `use*`, especially SWR hooks in `apps/web/lib/swr`.
- Prisma models use domain language like workspace/project, link, domain, partner, program, commission, payout, webhook.

Where type safety is strong:

- Shared packages.
- Zod-validated API boundaries.
- Prisma model interactions.
- OpenAPI-linked schemas.

Where type safety is weaker:

- `apps/web` app code because `strict` is disabled.
- Auth/provider callbacks where external profile shapes vary.
- Integration payloads before schema parsing.
- Places using direct imports from package internals, such as `@dub/utils/src/constants`.

## 7. Testing Strategy

Test frameworks:

- Vitest for app/API/domain tests: `apps/web/vitest.config.ts`.
- Playwright for E2E browser tests: `apps/web/playwright.config.ts`.
- Jest for Stripe UI extension tests: `packages/stripe-app/jest.config.js`.

Vitest organization:

- Tests live under `apps/web/tests`.
- Setup file: `apps/web/tests/setupTests.ts`.
- Feature folders include `links`, `analytics`, `tracks`, `partners`, `workspaces`, `webhooks`, `workflows`, `domains`, `commissions`, `campaigns`, `folders`, and `misc`.
- Helpers live in `apps/web/tests/utils`.
- Integration env schema is in `apps/web/tests/utils/env.ts`.

Vitest command:

```bash
pnpm --filter web test
```

Playwright organization:

- Tests live under `apps/web/playwright`.
- Partner tests: `apps/web/playwright/partners`.
- Workspace tests: `apps/web/playwright/workspaces`.
- Auth setup files create storage states in `playwright/.auth`.
- Local docs: `apps/web/playwright/README.md`.

Playwright command:

```bash
pnpm --filter web test:e2e
```

CI patterns:

- `.github/workflows/playwright.yaml` starts MySQL, MailHog, Redis, serverless Redis HTTP, and PlanetScale simulator, then runs Prisma, seeds Playwright data, builds web, and runs E2E.
- `.github/workflows/e2e.yaml` runs public API tests against a deployed URL.
- `.github/workflows/prettier.yaml` runs formatting checks.

How to add a new test:

1. For API/domain behavior, add `apps/web/tests/<area>/<name>.test.ts`.
2. Reuse helpers from `apps/web/tests/utils`.
3. If the test hits public API routes, use the existing integration harness and required `E2E_*` env variables.
4. For browser flows, add a spec under `apps/web/playwright/partners` or `apps/web/playwright/workspaces`.
5. Add a new Playwright project only if the test needs a different auth state, base URL, or dependency.

Notable gaps and risks:

- Many tests are integration-heavy and require realistic external/local services.
- Short-link middleware behavior has many branches; regression risk is high.
- Integration webhooks depend on external payload contracts and secrets.
- Full local E2E setup is heavier than unit tests.

## 8. Explanation For A Junior Developer

Start with the product idea: a user creates a short link in the dashboard or API. Someone clicks that short link. Dub redirects the visitor to the final URL and records analytics. Workspaces, domains, tags, partners, and webhooks are features around that core loop.

Read these files first:

1. `README.md` for the project overview.
2. `package.json` and `apps/web/package.json` for commands.
3. `apps/web/middleware.ts` to understand routing.
4. `apps/web/app/api/links/route.ts` to see a typical API route.
5. `apps/web/lib/api/links/create-link.ts` to see a domain service.
6. `apps/web/lib/middleware/link.ts` to understand click redirects.
7. `apps/web/lib/zod/schemas/links.ts` to understand validation.
8. `packages/prisma/schema/link.prisma` for the link database model.

Concepts to learn before contributing:

- Next.js App Router: `layout.tsx`, `page.tsx`, `route.ts`.
- Server components vs `"use client"` components.
- Zod parsing.
- Prisma models and queries.
- SWR data fetching.
- NextAuth basics.
- Redis cache vs database persistence.
- Why `waitUntil` is used for async side effects.

Common mistakes to avoid:

- Putting business logic directly in a route handler instead of `apps/web/lib/api`.
- Changing a Zod schema without thinking about API compatibility and OpenAPI docs.
- Forgetting that short-link redirect code is high traffic and very branchy.
- Writing UI mutation code without checking existing `next-safe-action` patterns.
- Assuming all app TypeScript is fully strict.
- Bypassing `withWorkspace` permissions on workspace API routes.
- Updating Prisma schema without running generation and considering migrations/push.

Suggested 3-day onboarding path:

Day 1:

- Run `pnpm install`.
- Read `README.md`, `apps/web/.env.example`, `apps/web/package.json`, and `apps/web/middleware.ts`.
- Start the local services and run Prisma generate/push if possible.
- Trace one link creation request from `apps/web/app/api/links/route.ts` into `apps/web/lib/api/links`.

Day 2:

- Read the dashboard route for links: `apps/web/app/app.dub.co/(dashboard)/[slug]/links/page.tsx` and `page-client.tsx`.
- Read `apps/web/lib/swr/use-links.ts`.
- Read tests in `apps/web/tests/links`.
- Make a tiny local-only experiment or add a small test to understand the harness.

Day 3:

- Trace a short-link click through `apps/web/lib/middleware/link.ts`.
- Read `packages/prisma/schema/link.prisma`, `domain.prisma`, and `workspace.prisma`.
- Pick a small bug, copy, validation, or test improvement.

## 9. Explanation For A Mid-Level Developer

The main architectural tradeoff is that Dub keeps a large product surface in one Next.js app while extracting reusable/publishable pieces into workspace packages. This keeps domain behavior close to routes and UI, but it means `apps/web/lib` is broad and requires discipline around boundaries.

Key extension points:

- Add API resources under `apps/web/app/api/<resource>/route.ts`, backed by `apps/web/lib/api/<resource>`.
- Add validation under `apps/web/lib/zod/schemas`.
- Add dashboard UI under `apps/web/app/app.dub.co/(dashboard)` and reusable feature UI under `apps/web/ui`.
- Add shared utilities to `packages/utils` only when they are truly cross-package.
- Add reusable UI primitives to `packages/ui` only when they are not app-domain-specific.
- Add database models in `packages/prisma/schema/*.prisma`.
- Add background work through `apps/web/lib/cron`/QStash and cron routes.

Performance considerations:

- `apps/web/lib/middleware/link.ts` is latency-sensitive. Prefer cache-first reads and async side effects.
- Link resolution uses Redis cache and PlanetScale edge reads.
- Click recording and webhook-like effects should use `ev.waitUntil` or QStash when response latency matters.
- Large workspaces have special search behavior: `MEGA_WORKSPACE_LINKS_LIMIT` influences link search mode in `apps/web/app/api/links/route.ts`.
- Analytics queries are Tinybird-backed, not regular Prisma reads.

Reliability concerns:

- Cache/database/Tinybird consistency is eventual in several flows.
- Webhook delivery depends on QStash and callback handling.
- Integration payloads must be schema-validated.
- Auth wrappers are security boundaries, not convenience helpers.
- Payment, partner, and enterprise paths are spread across `app/api`, `app/(ee)/api`, and domain libraries.

Where to look before making changes:

- API behavior: route file, corresponding `lib/api` folder, Zod schema, OpenAPI path, tests.
- UI behavior: App Router page, `page-client.tsx`, `apps/web/ui`, SWR hook.
- Database behavior: Prisma model, service function, tests.
- Redirect behavior: `middleware.ts`, `lib/middleware/link.ts`, link cache helpers.
- Auth behavior: `lib/auth/workspace.ts`, `lib/auth/session.ts`, `lib/auth/options.ts`.
- Event behavior: `lib/tinybird`, `packages/tinybird`, tests under `tests/tracks` and `tests/analytics`.

Suggested first meaningful PR:

Add a small link-management API enhancement, such as a new filter or response field that is already present in the database. This forces you to touch the right layers without boiling the ocean: schema, route/service, response transform, OpenAPI metadata, and tests.

## 10. How To Add A New Feature

Example feature: add an optional `notes` field to links so workspace users can store internal notes that are not shown to visitors.

Step 1: Start reading.

- `apps/web/app/api/links/route.ts`
- `apps/web/lib/api/links/process-link.ts`
- `apps/web/lib/api/links/create-link.ts`
- `apps/web/lib/api/links/utils/transform-link.ts`
- `apps/web/lib/zod/schemas/links.ts`
- `packages/prisma/schema/link.prisma`
- `apps/web/tests/links/create-link.test.ts`
- `apps/web/tests/links/update-link.test.ts`

Step 2: Decide the data model.

- Add `notes String?` to the Prisma `Link` model in `packages/prisma/schema/link.prisma`.
- Run Prisma format/generate.
- Consider whether notes should be included in public API responses, dashboard-only responses, or both.

Step 3: Update validation.

- Add `notes` to the create/update link Zod schemas in `apps/web/lib/zod/schemas/links.ts`.
- Enforce length limits with Zod.
- Add OpenAPI metadata if the field is public API-visible.

Step 4: Update processing and persistence.

- Ensure `processLink` accepts and normalizes `notes`.
- Ensure `createLink` writes `notes`.
- Update update-link services if links can be edited.
- Update `transformLink` if the API should return `notes`.

Step 5: Update UI.

- Find the link create/edit UI under `apps/web/ui/links` and dashboard routes under `apps/web/app/app.dub.co/(dashboard)/[slug]/links`.
- Add the field only where it belongs. Do not expose internal notes on public inspect/proxy pages.

Step 6: Add tests.

- Add API create/update tests in `apps/web/tests/links`.
- Test max length validation.
- Test that notes round-trip for authorized workspace users.
- Test that public/anonymous surfaces do not leak notes if that is a requirement.

Step 7: Validate locally.

```bash
pnpm --filter web prisma:generate
pnpm --filter web test -- links
pnpm --filter web lint
```

Step 8: Edge cases.

- Empty string vs `null`.
- Bulk create/update behavior.
- Import/export behavior.
- API compatibility for old clients.
- Whether notes should appear in webhooks.
- Whether notes should be searchable.

Review checklist:

- Auth and permissions are unchanged or explicitly justified.
- Zod schema, Prisma schema, and response transform agree.
- OpenAPI docs are updated if public.
- Tests cover create/update/read behavior.
- No sensitive/internal field leaks to public short-link pages.
- Cache invalidation is handled if cached link shape changes.

## 11. Senior Engineer Notes

Architectural strengths:

- Clear monorepo shape with Turborepo and pnpm workspaces.
- High-value shared packages: `@dub/prisma`, `@dub/ui`, `@dub/utils`, `@dub/email`.
- Route handlers are usually thin and delegate to domain services.
- Zod schemas define validation and documentation metadata.
- Short-link runtime is optimized around cache-first reads and async event recording.
- Test coverage is organized around product domains.

Areas of complexity:

- `apps/web/lib` is large and multi-domain; naming and locality matter.
- Middleware is a critical traffic hub.
- Link creation has many side effects: database, cache, Tinybird, webhooks, usage events, storage, QStash, A/B tests.
- Auth combines sessions, API keys, RBAC, token scopes, SAML, credentials, OAuth, and plan checks.
- Enterprise route groups and commercial/open-core boundaries require care.
- External integrations create many payload and retry edge cases.

Technical debt or risky areas:

- `apps/web` has `strict: false`, so code review must watch nullability and external payload types.
- Some imports reach into package internals, such as `@dub/utils/src/constants`.
- Several flows rely on eventual consistency between Prisma, Redis, and Tinybird.
- Local development requires many services and secrets for full fidelity.
- Some auth and provider callback typing uses `any`, `@ts-ignore`, or provider-specific assumptions.

Implicit conventions:

- Put reusable business logic in `apps/web/lib/api`, not directly in route files.
- Use `withWorkspace`, `withSession`, or `withAdmin` instead of rolling custom auth.
- Add schemas before adding route behavior.
- Use `waitUntil` or QStash for non-blocking side effects.
- Keep app-specific UI in `apps/web/ui`; move only broadly reusable primitives to `packages/ui`.
- Treat Zod schema changes as API contract changes.

Places to be careful:

- `apps/web/middleware.ts`
- `apps/web/lib/middleware/link.ts`
- `apps/web/lib/auth/workspace.ts`
- `apps/web/lib/auth/options.ts`
- `apps/web/lib/api/links/create-link.ts`
- `apps/web/lib/webhook/*`
- `packages/prisma/schema/*.prisma`
- `apps/web/lib/zod/schemas/*`

## 12. Glossary

Dub
: The product and platform in this repository.

Short link
: A compact URL on a Dub or custom domain that redirects to a destination URL.

Workspace / Project
: The tenant/account boundary. Some older code and Prisma models use `project` while user-facing language says workspace.

Domain
: A hostname used for short links, app routing, API routing, admin, or partners.

Key
: The path portion of a short link, for example `abc` in `dub.sh/abc`.

Link cache
: Redis-backed cached representation of link data used by redirect middleware.

Tinybird
: Analytics/event storage and query backend for clicks, leads, sales, API logs, and reporting.

PlanetScale
: MySQL-compatible serverless database used with Prisma.

Prisma
: ORM and generated database client, packaged in `@dub/prisma`.

QStash
: Upstash queue/scheduler used for background jobs, delayed work, and webhook delivery.

SWR
: Client-side data fetching/cache library used by dashboard components.

NextAuth
: Authentication library used for email, OAuth, SAML, credentials, and session handling.

SAML / BoxyHQ
: Enterprise single sign-on integration.

Webhook
: Customer-configured outbound event delivery for events like link created or conversion tracked.

Conversion
: Tracked business event after a click, commonly lead or sale.

Partner / Program
: Affiliate/partner marketing concepts. A program belongs to a workspace; partners enroll and generate attributed links/conversions.

Open Core / `(ee)`
: Dub is mostly AGPL open source, with enterprise features grouped under `(ee)` route groups and related code.

Inspect mode
: Appending `+` to a short link key can show an inspect page instead of immediately redirecting, when allowed.

Root link
: A link represented internally with key `_root`, used for root-domain behavior.
