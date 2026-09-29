# Runtime and Tooling Map

## Toolchain

| Tool | Config | Notes |
| --- | --- | --- |
| pnpm 9 | `packageManager` in [package.json](../../../package.json) | workspace protocol (`workspace:*`) links internal packages |
| Turborepo | [turbo.json](../../../turbo.json) | `build` depends on `^build` (topological); `test` depends on `^build`; `dev` uncached/persistent |
| Next.js (App Router) | [apps/web/next.config.js](../../../apps/web/next.config.js) | dev uses `--turbopack` on port **8888** |
| Prisma | [packages/prisma](../../../packages/prisma) | schema split across 36 files in `schema/`; `prisma:push` (db push) — no migrations directory found in the package (**inferred**: schema-push workflow, PlanetScale-style; verify against their deploy docs before claiming) |
| Vitest | [apps/web/vitest.config.ts](../../../apps/web/vitest.config.ts) | `dir: ./tests`, 50s timeout, env loaded via `loadEnv` — integration tests |
| Playwright | [apps/web/playwright.config.ts](../../../apps/web/playwright.config.ts) | browser E2E |
| Prettier | [prettier.config.js](../../../prettier.config.js) | plugins: organize-imports + tailwindcss — imports and classes auto-sorted; don't fight it |
| ESLint | `next lint` | no standalone typecheck script (see ticket 16) |

## Where code executes (the boundary map)

| Boundary | What runs there | Sharp edge |
| --- | --- | --- |
| **Middleware (Node runtime)** | hostname dispatch + full redirect engine — [middleware.ts#L21-L22](../../../apps/web/middleware.ts#L21-L22) explicitly sets `runtime: "nodejs"` | middleware traditionally ran on the Edge runtime; this repo opted into Node — full API surface, but per-instance state (LRU) now matters |
| **Server (RSC / route handlers)** | API routes, server components, server actions | `server-only` import guards (e.g. [errors.ts#L2](../../../apps/web/lib/api/errors.ts#L2)) prevent client bundling |
| **Client (browser)** | `"use client"` components, SWR hooks | anything imported there ships in the bundle — watch `@dub/utils` imports |
| **Vercel platform** | `waitUntil`, geolocation headers, runtime cache (`getCache()`) | `process.env.VERCEL === "1"` gates real geo vs `LOCALHOST_GEO_DATA` ([record-click.ts#L118-L129](../../../apps/web/lib/tinybird/record-click.ts#L118-L129)) — local behavior ≠ prod behavior |
| **QStash (external)** | webhook delivery, cron fan-out, delayed jobs | signature verification **skipped locally** ([verify-qstash.ts#L18-L21](../../../apps/web/lib/cron/verify-qstash.ts#L18-L21)) |
| **Tinybird (external)** | analytics SQL (pipes) | query logic lives in [packages/tinybird](../../../packages/tinybird), not in app code — "where is this aggregation computed?" answer: not in JS |

## Environment variables (high level, no secrets)

Groups you'll see referenced: `DATABASE_URL` (Prisma/PlanetScale), `UPSTASH_*`/`QSTASH_*` (Redis, queue, signing keys), `TINYBIRD_API_*` ([record-click.ts#L181-L186](../../../apps/web/lib/tinybird/record-click.ts#L181-L186)), `NEXTAUTH_*` (sessions), storage (R2), `VERCEL` (platform detection), `E2E_BASE_URL`/`E2E_TOKEN` (tests — [tests/utils/env.ts](../../../apps/web/tests/utils/env.ts)). Turbo declares `**/.env` a global dependency ([turbo.json](../../../turbo.json)) so env changes bust build caches.

## Commands (all __inferred__ — see [command cheatsheet](../09-reference/command-cheatsheet.md))

```bash
pnpm dev                    # root: turbo dev (all packages)
cd apps/web && pnpm dev     # app only: Next on :8888 + Prisma Studio
pnpm build:packages         # build shared packages once after clone
cd apps/web && pnpm test    # vitest integration — needs E2E_* env
cd apps/web && pnpm generate-openapi   # regenerate the OpenAPI spec after schema changes
```

Interview angle: "walk me through your monorepo setup" is a real screen question. The strong answer names: workspace protocol, topological builds with caching (`^build`), where contracts live (schemas package-adjacent), and one sharp edge (env-dependent cache busting, or Node-vs-Edge middleware choice).

Drill: from [turbo.json](../../../turbo.json) alone, explain what happens (build-wise) when you change one line in `packages/utils` and run `pnpm build`.
