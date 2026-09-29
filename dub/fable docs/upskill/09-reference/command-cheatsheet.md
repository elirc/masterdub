# Command Cheatsheet

All commands are __inferred__ from manifests unless marked __verified__ (nothing was executed during authoring — no `.env`/services on the authoring machine; see [verification-log.md](verification-log.md)). Sources: [package.json](../../../package.json), [apps/web/package.json](../../../apps/web/package.json), [turbo.json](../../../turbo.json).

## Setup

| Command | Where | Notes |
| --- | --- | --- |
| `pnpm install` | root | pnpm 9.15.9 (`packageManager` field enforces via corepack) |
| `pnpm build:packages` | root | build all `packages/*` once so the app resolves workspace deps |
| `docker compose up -d` | apps/web | local backing services per [docker-compose.yml](../../../apps/web/docker-compose.yml) — inspect it first for what it provides |

## Develop

| Command | Where | Notes |
| --- | --- | --- |
| `pnpm dev` | root | turbo dev across workspace (persistent, uncached) |
| `pnpm dev` | apps/web | Next.js (turbopack) on **:8888** + Prisma Studio, both via concurrently; runs `prisma:generate` first |
| `pnpm lint` | root or apps/web | `next lint` in the app — does not fully typecheck |
| `npx tsc --noEmit` | apps/web | manual typecheck; no script exists (see [ticket 16](../06-contribution-practice/01-good-first-tickets.md)) |
| `pnpm format` / `pnpm prettier-check` | root | prettier with organize-imports + tailwind plugins |

## Test

| Command | Where | Notes |
| --- | --- | --- |
| `pnpm test` | apps/web | vitest, `-no-file-parallelism --bail=1`, **needs `E2E_BASE_URL` + `E2E_TOKEN`** — tests hit a live deployment |
| `pnpm test tests/links/create-link.test.ts` | apps/web | single file |
| `pnpm test:e2e` / `test:e2e:ui` / `test:e2e:headed` | apps/web | Playwright |
| `pnpm test` | root | turbo test pipeline (depends on `^build`) |

## Database (Prisma / PlanetScale)

| Command | Where | Notes |
| --- | --- | --- |
| `pnpm prisma:generate` | apps/web | regenerate client after any schema edit — stale client = stale types |
| `pnpm prisma:push` | apps/web | db push (schema state sync; no migration files) — dev workflow; prod process unverified |
| `pnpm prisma:studio` | apps/web | data browser (auto-started by `dev`) |
| `pnpm prisma:format` | apps/web | format the 36 `.prisma` files |

## Build & codegen

| Command | Where | Notes |
| --- | --- | --- |
| `pnpm build` | root | turbo build, topological + cached; env changes bust caches (`globalDependencies: ["**/.env"]`) |
| `pnpm build --filter=web` | root | app only |
| `pnpm generate-openapi` | apps/web | regenerate OpenAPI from Zod schemas — run after any public-contract change |
| `pnpm script <name>` | apps/web | run one-off scripts via `tsx ./scripts/run.ts` |

## Useful non-package commands (__verified__ shapes, generic tools)

| Command | Purpose |
| --- | --- |
| `rg "requiredPermissions" apps/web/app/api -l` | audit which routes declare permissions |
| `rg "conn.execute" apps/web/lib` | find raw-SQL sites before schema changes |
| `rg "waitUntil\(" apps/web/lib -l` | inventory fire-and-forget side effects |
| `rg "process.env.VERCEL" apps/web` | find env-conditional behavior |
| `curl -sI https://dub.sh/github` | observe a real redirect's status/headers (302, cookies) |
