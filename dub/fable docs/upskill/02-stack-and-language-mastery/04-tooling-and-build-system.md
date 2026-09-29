# Tooling and Build System

## Mental model

A monorepo is a dependency graph with a task runner on top. Three questions locate any build problem: (1) which package owns the file, (2) who depends on it (`workspace:*` edges), (3) what task pipeline runs over it ([turbo.json](../../../turbo.json) — `build` is topological via `dependsOn: ["^build"]`, cached by input hashes; `dev` is persistent and uncached).

## Repo specifics worth knowing cold

- **Workspace globs**: `apps/*`, `packages/*`, `packages/embeds/*`, and the odd one — `apps/web/.react-email` ([pnpm-workspace.yaml](../../../pnpm-workspace.yaml)). Nested workspace packages exist; `ls` isn't enough to enumerate them.
- **Published packages** (`publish-*` scripts in [package.json](../../../package.json)): `@dub/cli`, `@dub/embed-core`, `@dub/embed-react`, `@dub/prisma`, `@dub/tailwind-config`, `@dub/ui`, `@dub/utils`. Their exports are semver contracts — an "internal" refactor of `packages/utils` can be a breaking release.
- **Prisma generate before everything**: `dev`, `build`, and `test` in [apps/web/package.json](../../../apps/web/package.json) all run `prisma:generate` first — the Prisma client is *generated code*; a schema change without regeneration produces stale types (classic monorepo gotcha).
- **Env as build input**: `globalDependencies: ["**/.env"]` in turbo.json — changing env busts caches; conversely, env *not* in that glob can produce stale-cache confusion in other repos. Know this pattern.
- **Formatting is automated opinion**: prettier with `organize-imports` and `tailwindcss` plugins ([prettier.config.js](../../../prettier.config.js)) — import order and class order are machine-owned; hand-sorting is wasted diff.
- **`resolutions`**: the root pins `chrono-node: 2.7.5` — a transitive-dependency override; the mechanism to know when a sub-dependency breaks you.
- **Monorepo workaround plugins**: `@prisma/nextjs-monorepo-workaround-plugin` in web's deps — evidence that bundling generated native clients across workspace boundaries is a real problem class, solved here with a plugin rather than convention.

## Failure modes to recognize (any monorepo)

- Stale generated code (Prisma client, OpenAPI) after schema edits — fix: generation wired into every consuming script, as here.
- Phantom dependencies: importing a package you never declared, which pnpm's strict node_modules makes loud (a *feature* — npm/yarn hoisting hides it).
- Task-graph gaps: a package whose `build` isn't in the pipeline builds locally (via editor TS) but fails in CI.
- Cache poisoning: an undeclared input (env var, network fetch at build time) makes cached builds wrong; turbo's answer is declaring inputs.

## Drills

1. Trace what `pnpm build --filter='@dub/cli'` builds, using turbo's `^build` semantics.
2. You edit `packages/utils/src/constants`. List every consumer that must rebuild, and how turbo knows.
3. Explain why `apps/web`'s `test` script runs `prisma:generate` even though tests hit a deployed API (hint: the tests import `@dub/prisma/client` *types* — [create-link.test.ts#L3](../../../apps/web/tests/links/create-link.test.ts#L3)).

## Interview angle

- "Describe your team's build/CI setup" — screen-round staple; answer with the graph model + caching + one gotcha you can narrate (stale Prisma client is a great one).
- "How do you share code between apps?" → workspace packages with explicit publish boundaries; contrast with copy-paste and with a shared `common/` dumping ground.
- Cross-links: [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) debugging round 4 uses a build-related scenario.
