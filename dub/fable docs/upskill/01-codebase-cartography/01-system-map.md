# System Map

## Monorepo shape

pnpm workspaces + Turborepo ([pnpm-workspace.yaml](../../../pnpm-workspace.yaml), [turbo.json](../../../turbo.json)). One product app, many shared packages — several of which are **published to npm** (see the `publish-*` scripts in [package.json](../../../package.json)), which makes their public exports *contracts*, not internals.

```
dub/
├── apps/
│   └── web/                  # the entire product: dashboard + API + redirect engine (Next.js App Router)
└── packages/
    ├── prisma/               # schema (36 .prisma files) + client singletons — published as @dub/prisma
    ├── ui/                   # shared React components — published as @dub/ui
    ├── utils/                # constants + pure functions — published as @dub/utils
    ├── email/                # react-email templates
    ├── embeds/{core,react}   # customer-embeddable widgets — published
    ├── cli/                  # @dub/cli
    ├── tinybird/             # Tinybird pipes/datasources (analytics SQL lives here, not in apps/web)
    ├── stripe-app/, hubspot-app/  # marketplace integrations
    └── tailwind-config/, tsconfig/
```

## Runtime surfaces (one app, five faces)

The single Next.js app serves different products depending on **hostname**, dispatched in [middleware.ts#L35-L90](../../../apps/web/middleware.ts#L35-L90):

| Hostname | Middleware | Surface |
| --- | --- | --- |
| `app.dub.co` | `AppMiddleware` | dashboard UI ([app/app.dub.co/](../../../apps/web/app/app.dub.co/)) |
| `api.dub.co` | `ApiMiddleware` | public REST API (rewrites into [app/api/](../../../apps/web/app/api/)) |
| `admin.dub.co` | `AdminMiddleware` | internal admin |
| `partners.dub.co` | `PartnersMiddleware` | partner-program portal ([app/(ee)](../../../apps/web/app/(ee))) |
| everything else | `LinkMiddleware` | **redirect engine** — any customer domain |

That last row is the key architectural fact: *every unknown hostname is assumed to be a customer's short-link domain.*

## Ownership map

| Concern | Lives in | Notes |
| --- | --- | --- |
| UI pages | `apps/web/app/app.dub.co/` | route groups: `(auth)`, `(dashboard)`, `(onboarding)`… |
| UI components | `apps/web/ui/` (feature-specific) + `packages/ui` (generic) | the boundary between them is "does another feature need it" |
| API routes | `apps/web/app/api/` + `apps/web/app/(ee)/api/` | thin handlers; real logic in `lib/api` |
| Domain logic | `apps/web/lib/api/<resource>/` | e.g. [lib/api/links/](../../../apps/web/lib/api/links/) — the heart |
| Auth wrappers | `apps/web/lib/auth/` | [workspace.ts](../../../apps/web/lib/auth/workspace.ts) et al |
| Contracts | `apps/web/lib/zod/schemas/` | 70+ schema files; also feed OpenAPI |
| Server actions | `apps/web/lib/actions/` | next-safe-action clients ([safe-action.ts](../../../apps/web/lib/actions/safe-action.ts)) |
| Client data hooks | `apps/web/lib/swr/` | one `use-<resource>` per API resource |
| Persistence | `packages/prisma/schema/` + `apps/web/lib/planetscale/` | Prisma for CRUD, HTTP driver for hot paths |
| Analytics events | `apps/web/lib/tinybird/` + `packages/tinybird/` | record-* writers; pipes in the package |
| Cache / queues | `apps/web/lib/upstash/` + `lib/webhook/qstash.ts` + `lib/cron/` | Redis, Redis streams, QStash |
| Background jobs | `apps/web/app/(ee)/api/cron/` | 38 job directories, invoked by QStash/Vercel cron |
| Tests | `apps/web/tests/` (vitest integration) + `apps/web/playwright/` | hit a live deployment |

## Public interfaces vs private internals

Public (changing these is a breaking change): REST API shapes (`lib/zod/schemas` → OpenAPI), webhook payloads ([lib/webhook/schemas.ts](../../../apps/web/lib/webhook/schemas.ts)), published packages' exports, the `(ee)` embed tokens.
Private (refactor freely with tests): everything in `lib/api/*` function signatures, UI components, SWR hooks.

`(ee)` directories are the enterprise/commercial edition — check [LICENSE.md](../../../LICENSE.md) before assuming AGPL applies uniformly.

## Request lifecycles at a glance

```
Click:    browser → middleware(hostname) → LinkMiddleware → cache/DB → 302
                                                     └─ waitUntil → recordClick → Tinybird/MySQL/Redis/QStash
API:      client → ApiMiddleware(rewrite) → route.ts → withWorkspace → handler → lib/api/* → prisma
                                                                          └─ waitUntil → cache/webhooks/logs
Dashboard: browser → AppMiddleware(auth redirect) → RSC page → client components → SWR → /api/*
Cron:     QStash → app/(ee)/api/cron/* → verifyQstashSignature → job
```

Drill: without looking, name which middleware handles `dub.sh/stats/github`, `go.customer.com/promo`, and `api.dub.co/links` — then check [middleware.ts#L35-L90](../../../apps/web/middleware.ts#L35-L90). (Watch out: the first one is a rewrite *before* the admin/partner checks.)
