# Verification Log

Running record of what was inspected while authoring this curriculum. Anchors in the docs were confirmed against the files listed here on the dates shown.

**Environment:** Windows 11, working tree at `masterdub/dub`, no `.env` present at `apps/web` (services: PlanetScale MySQL, Upstash Redis/QStash, Tinybird — none available locally). No dev server was started; no tests were executed. All runtime behavior claims are derived from reading code, not from observing execution — treat behavior claims as high-confidence static analysis, not observation.

## 2026-07-09 — initial exploration and authoring

| What | How | Result / notes |
| --- | --- | --- |
| Monorepo layout | `ls`, read `package.json`, `pnpm-workspace.yaml`, `turbo.json` | pnpm@9.15.9 + Turborepo; workspaces: `apps/*`, `packages/*`, `packages/embeds/*`, `apps/web/.react-email` |
| Web app scripts | read `apps/web/package.json` | `dev` runs Next on port 8888 + prisma studio via concurrently; `test` = vitest (`-no-file-parallelism --bail=1`); `test:e2e` = playwright |
| Middleware routing | read `apps/web/middleware.ts` (full, 91 lines) | hostname-based dispatch: App/API/Admin/Partners middlewares, else LinkMiddleware; `runtime: "nodejs"` |
| Redirect flow | read `apps/web/lib/middleware/link.ts` (full, 581 lines) | key normalization, LRU/Redis/DB lookup, password/expired/banned branches, device/geo targeting, `ev.waitUntil(recordClick(...))` |
| Auth wrapper | read `apps/web/lib/auth/workspace.ts` (full, 528 lines) | `withWorkspace`: Bearer token vs session, token cache, plan-based rate limits, RBAC permissions, plan/role/feature-flag gates |
| Links API | read `apps/web/app/api/links/route.ts` (full, 111 lines) | GET/POST with `requiredPermissions`; anonymous creation path rate-limited 10/day/IP |
| Link validation | read `apps/web/lib/api/links/process-link.ts` (lines 1–120 of ~800+) | returns `{link,error,code}` union instead of throwing; plan-gated features |
| Link creation | read `apps/web/lib/api/links/create-link.ts` (full, 238 lines) | prisma create with retry; `waitUntil` fan-out: Redis cache, Tinybird recordLink, R2 image upload, QStash delayed delete for anonymous links, usage events |
| Click recording | read `apps/web/lib/tinybird/record-click.ts` (full, 359 lines) | bot filter, 1-hour dedup via Redis, EU IP redaction, `Promise.allSettled` fan-out (Tinybird HTTP, raw SQL counters, Redis streams w/ DB fallback), webhook dispatch |
| Link cache | read `apps/web/lib/api/links/cache.ts` (lines 1–100 of ~200) | 3 tiers: in-process LRU (10k, 5s TTL) → Upstash Redis (24h) → Vercel runtime cache (5m) |
| Webhook delivery | read `apps/web/lib/webhook/qstash.ts` (full, 135 lines) | QStash publish with HMAC `Dub-Signature`, success/failure callback URLs; TODO notes missing deduplicationId |
| Analytics API | read `apps/web/app/api/analytics/route.ts` (full, 139 lines) | folder-level authz via `verifyFolderAccess`, plan-based date-range gate, deprecated-endpoint compatibility |
| Prisma schema | read `packages/prisma/schema/link.prisma` (full); listed 36+ `.prisma` files | Link model: denormalized counters (clicks/leads/sales), composite uniques `(domain,key)`, `(projectId,externalId)`, 8 explicit indexes |
| RBAC | read `apps/web/lib/api/rbac/permissions.ts` (lines 1–60) | const-array `PERMISSION_ACTIONS`, role→permission table (owner/member/viewer/billing) |
| Error handling | read `apps/web/lib/api/errors.ts` (lines 1–130) | `DubApiError` class + `handleAndReturnErrorResponse`; Zod errors → 422; Prisma P2025 → 404 |
| Server actions | read `apps/web/lib/actions/safe-action.ts` (lines 1–60) | next-safe-action clients: `actionClient` → `authUserActionClient` → `authActionClient` (workspace membership) |
| SWR hooks | read `apps/web/lib/swr/use-links.ts` (lines 1–50); listed ~60 hooks in `lib/swr/` | `useLinks` builds querystring from filters; mega-workspace special-casing |
| Tests | read `apps/web/tests/utils/integration.ts` (full), `tests/links/create-link.test.ts` (lines 1–80), `vitest.config.ts` | integration tests hit a live deployment (`E2E_BASE_URL` + `E2E_TOKEN`) — not unit tests; playwright config also present |
| Cron/QStash verify | read `apps/web/lib/cron/verify-qstash.ts` (lines 1–40); listed `app/(ee)/api/cron/*` (38 job dirs) | signature verification skipped when `VERCEL !== "1"` |
| Track lead route | read `app/(ee)/api/track/lead/route.ts` (lines 1–30) | conversion tracking behind `withWorkspace`; deprecated field back-compat |
| Existing docs | noted `CODEBASE_ONBOARDING_GUIDE.md`, `SUBAGENT_REBUILD_PLAYBOOK.md` at repo root | pre-existing AI-authored guides; this curriculum is independent and does not overwrite them |

## Commands run

| Command | Result |
| --- | --- |
| `ls` / `find` / `grep -n` over repo | success (structure and line numbers as recorded above) |
| `git remote -v` (repo root) | remote points at `github.com/elirc/grouphelpdesk.git` — the outer git repo is the user's home directory, **not** a dub clone; `dub/` itself has no own `.git` visible from `masterdub` (dub history not inspected) |
| `pnpm install` / `pnpm dev` / `pnpm test` | **NOT run** (no `.env`, no backing services). All such commands in the docs are marked __inferred__ |

## Post-authoring anchor spot-checks (2026-07-09)

Re-confirmed via grep/ls after all docs were written: `LINKS_MAX_PAGE_SIZE` at `lib/zod/schemas/links.ts:36`; `lib/axiom/server.ts` exists; `app/api/webhooks/callback/` exists; `get-identity-hash.ts`, `with-prisma-retry.ts`, `lib/folder/permissions.ts`, `lib/webhook/failure.ts`, `get-link-or-throw.ts` all exist at the cited paths. Final tree: 53 markdown files under `fable docs/upskill/` (originally authored as `docs/upskill/`, moved 2026-07-09).

## 2026-10-06 — accuracy pass against the committed snapshot

Re-checked against this repository (`elirc/masterdub`, commit `64dba44`), statically — nothing was installed or run:

- The `git remote` row above is stale: the curriculum now lives in its own repository, `elirc/masterdub`, with `dub/` as a plain folder (no upstream Dub history).
- All 1,011 relative Markdown links under `fable docs/upskill/` were resolved. `00-fast-track.md` linked one directory too shallow (`../` instead of `../../`) and two `04-code-reading-gym` pages one too deep; both fixed. Every `#Lx-Ly` anchor is within its file's length, and the fast-track anchors (redirect guards, QStash skip, `withWorkspace`, links route) were spot-read and match.
- `apps/web/scripts/generate-openapi.ts` is named by the `generate-openapi` script in `apps/web/package.json` but is **not in this snapshot**; links to it were replaced with plain text.
- `apps/web/lib/zod/schemas/` holds 65 files (docs said "70+"); corrected.
- File sizes in the table above have drifted slightly: `process-link.ts` is 591 lines (not "~800+"), `lib/swr/` has 103 files (not "~60 hooks"), `cache.ts` is 195 lines. The other line counts match to within one line.

## Known uncertainties / not covered

- Anything under `app/(ee)` beyond the files listed (partner programs, payouts, bounties, fraud) was surveyed by directory listing only — depth there is thinner.
- `process-link.ts` was read only through line ~120; claims about its later sections (key checks, folder access, program checks) come from its imports and call sites and are labeled accordingly.
- Tinybird pipe definitions live in `packages/tinybird` — not read; analytics query internals (`lib/analytics/get-analytics.ts`) not read in full.
- Prisma migrations directory: the schema uses `prisma:push` (db push) rather than a visible `migrations/` folder in `packages/prisma` — schema evolution strategy is inferred, marked as such in 03/02.
- No runtime observation: dev server, tests, and cron jobs were never executed.
