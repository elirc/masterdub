# Rebuilding Dub With Subagents: Efficiency Playbook

This document explains how to use subagents if someone wanted to recreate a Dub-like TypeScript project from scratch. The goal is not to clone every line, but to divide the work so architecture, implementation, verification, and documentation happen faster with fewer blind spots.

## Target Architecture

Recreate a monorepo with:

- A Next.js App Router web app.
- Host-aware middleware for app/API/admin/partner/short-link traffic.
- Workspace-aware API routes.
- Prisma-backed database package.
- Redis cache/rate-limit layer.
- Analytics event layer.
- QStash-style background jobs.
- Shared UI and utility packages.
- Zod schemas and OpenAPI generation.
- Vitest and Playwright tests.
- Optional CLI and embed packages.

## Recommended Subagent Roles

Use one lead engineer agent and several focused subagents.

Lead architect
: Owns decisions, interfaces, naming, code review, integration, and final docs.

Repo scaffold agent
: Creates Turborepo, pnpm workspace, shared tsconfig, lint/format/build scripts, root README, and package boundaries.

Database agent
: Designs Prisma schema, ID conventions, migrations/push workflow, database client exports, seed data, and test fixtures.

Routing/middleware agent
: Builds host-aware middleware, short-link parser, redirect/rewrite flow, cache lookup interface, and click ID cookie behavior.

API/domain agent
: Builds workspace API wrappers, Zod schemas, route handlers, domain services, error handling, and OpenAPI metadata.

Dashboard UI agent
: Builds App Router layouts, dashboard pages, client components, SWR hooks, and feature UI.

Auth/security agent
: Builds NextAuth config, session helpers, API key auth, workspace RBAC, token scopes, rate limits, and SSO extension points.

Analytics/events agent
: Builds click/lead/sale event interfaces, Tinybird-like client abstraction, analytics queries, and event tests.

Background jobs/webhooks agent
: Builds cron/QStash abstraction, webhook publisher, delivery callback handler, retry/failure records, and signature verification.

Testing agent
: Builds Vitest harness, Playwright projects, fixtures, env schema, CI workflow, and smoke tests.

Docs/onboarding agent
: Writes local setup docs, architecture guide, contribution guide, API docs, and glossary.

## Suggested Build Sequence

### Phase 1: Foundation

Run these in parallel:

- Repo scaffold agent: create monorepo, scripts, shared configs.
- Database agent: draft Prisma schema for User, Workspace, Domain, Link, Tag, Webhook, Event.
- Auth/security agent: draft auth model and permission matrix.
- Docs agent: start setup docs and architecture decision log.

Lead architect then reconciles names and boundaries.

### Phase 2: Core Link Loop

Run these in parallel with clear interfaces:

- Routing/middleware agent owns `apps/web/middleware.ts` and `lib/middleware/*`.
- API/domain agent owns `app/api/links` and `lib/api/links`.
- Database agent owns Prisma models and seed data.
- Analytics/events agent owns event interfaces and fake/local analytics adapter.

Shared contracts to define first:

```ts
type LinkRecord = {
  id: string;
  domain: string;
  key: string;
  url: string | null;
  workspaceId: string | null;
  expiresAt?: Date | null;
  disabledAt?: Date | null;
};

type RecordedClick = {
  clickId: string;
  linkId: string;
  domain: string;
  key: string;
  url: string | null;
  timestamp: string;
};
```

Integration target:

- `POST /api/links` creates a link.
- Visiting `/{key}` on a short-link host redirects.
- Click events are recorded asynchronously.
- Cache miss reads from database and populates cache.

### Phase 3: Dashboard And Workspace APIs

Run these in parallel:

- Dashboard UI agent: workspace shell, links list, create/edit modal.
- API/domain agent: workspace wrapper, links list/update/delete routes.
- Auth/security agent: session/API-key auth and RBAC checks.
- Testing agent: create API tests for links and workspace auth.

Lead architect should enforce:

- Routes stay thin.
- Zod schemas live with domain schemas.
- Business logic lives in `lib/api`.
- UI fetches through SWR hooks.

### Phase 4: External Systems

Run these in parallel:

- Background jobs/webhooks agent: outbound webhooks, queue abstraction, delivery callback.
- Analytics/events agent: real analytics backend integration.
- Auth/security agent: OAuth/SAML providers.
- Database agent: billing/partner/webhook schema expansion.

Add feature flags or adapters so local development can run without every vendor secret.

### Phase 5: Hardening

Run these in parallel:

- Testing agent: Playwright onboarding and link-management flows.
- API/domain agent: OpenAPI generation and response contract tests.
- Routing/middleware agent: redirect edge-case tests.
- Docs agent: onboarding guide and runbooks.

Lead architect then does final integration, threat-model pass, performance pass, and API compatibility pass.

## Subagent Task Templates

### Explorer Prompt

Use for read-only discovery:

```text
Inspect <path>. Do not edit files. Return concise notes with file paths about:
- architecture and boundaries
- entry points
- important types/functions
- dependencies and side effects
- risks and test gaps
```

### Worker Prompt

Use for implementation:

```text
You are responsible for <files/modules>. You are not alone in the codebase; do not revert edits made by others. Implement <feature> using existing patterns. Add focused tests. In your final response, list changed files, commands run, and remaining risks.
```

### Verification Prompt

Use for review:

```text
Review the changes for bugs, regressions, missing tests, security issues, and API compatibility. Do not edit files. Return findings first with file paths and line references.
```

## Parallelization Rules

Good parallel splits:

- Database schema vs UI shell.
- API schemas/services vs Playwright test scaffolding.
- Middleware redirect logic vs dashboard CRUD UI.
- Webhook delivery vs analytics queries.
- Docs vs implementation verification.

Avoid parallel edits to:

- The same route file.
- The same Prisma model.
- Shared schema files without a clearly assigned owner.
- Root package scripts unless one agent owns the repo scaffold.
- Auth wrappers, because many features depend on them.

## Interface-First Contracts

Before multiple workers implement, the lead should define:

- Workspace ID and slug semantics.
- Link key normalization rules.
- Error response shape.
- Zod schema naming.
- API wrapper context shape.
- Cache key format.
- Event payload shape.
- Webhook payload shape.
- Permission names.

These contracts prevent expensive merge conflicts and semantic drift.

## Minimal Viable Rebuild Milestones

Milestone 1: Monorepo boots.

- `pnpm install`
- `pnpm build`
- Next app renders a placeholder dashboard.

Milestone 2: Link creation works.

- Prisma schema has users/workspaces/domains/links.
- `POST /api/links` validates and creates a link.
- `GET /api/links` lists links for a workspace.

Milestone 3: Redirect path works.

- Middleware detects short-link host.
- Cache-first link lookup.
- Redirect response.
- Async click recording stub.

Milestone 4: Dashboard works.

- Sign in.
- Workspace shell.
- Links list.
- Create/edit/delete link.

Milestone 5: Production concerns.

- Rate limits.
- API keys.
- Webhooks.
- Analytics queries.
- Background jobs.
- E2E tests.
- OpenAPI docs.

## Suggested Verification Matrix

Run after major phases:

```bash
pnpm prettier-check
pnpm lint
pnpm build
pnpm test
pnpm --filter web test:e2e
```

Feature-specific checks:

- Create a link through API.
- Create a link through UI.
- Visit the short link and confirm redirect.
- Confirm click event recorded.
- Confirm permissions block unauthorized workspace access.
- Confirm Zod errors return the standard API error shape.
- Confirm webhook jobs enqueue and callback handling works.

## Lessons From This Repository

Useful patterns to copy:

- Keep route handlers thin and delegate domain behavior.
- Centralize auth/permission wrappers.
- Use Zod as both validation and documentation source.
- Use a cache-first redirect path.
- Push non-blocking side effects into `waitUntil`/queue work.
- Keep shared packages genuinely shared.
- Organize tests by product domain.

Risks to design around earlier:

- Full local setup can become service-heavy.
- Analytics and database consistency is often eventual.
- Auth wrappers become critical security infrastructure.
- Middleware changes have enormous blast radius.
- Public API schema changes need docs and tests.
- A large `lib` folder needs conventions before it grows.
