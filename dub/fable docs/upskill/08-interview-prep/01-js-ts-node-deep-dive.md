# JS / TS / Node Deep-Dive Question Cards

Fourteen cards. Practice: read the question, answer aloud for 90 seconds, *then* read the card. Every repo anchor is a place you can point to in an interview ("in a codebase I've studied deeply…"). Anchors verified 2026-07-09.

---

## Q1: What's the difference between `Promise.all`, `Promise.allSettled`, and `Promise.race` — and when have you actually needed `allSettled`?

Round: JS/TS deep-dive
What it's really testing: whether your async knowledge comes from shipping or from flashcards.
Repo anchor: [apps/web/lib/tinybird/record-click.ts#L177-L230](../../../apps/web/lib/tinybird/record-click.ts#L177-L230)
Junior answer sounds like: definitions of each.
Mid-level answer adds: a concrete case — recording a click fans out five *independent* writes (analytics ingest, two counters, cache set, stream publish); `all` would let one Redis blip cancel nothing (the others still run — `all` only rejects early, it doesn't cancel!) but would hide which succeeded; `allSettled` gives per-operation outcomes to log.
Senior answer includes: `Promise.all` does **not cancel** in-flight siblings on rejection — a common misconception; the real design question is which effects are allowed to partially succeed, and that `allSettled` without alerting is just organized silence.
Likely follow-ups: "How would you retry the failed ones?" (idempotency per operation); "What does `race` give you?" (timeouts — see `redisGlobalWithTimeout` in [cache.ts#L85](../../../apps/web/lib/api/links/cache.ts#L85), investigate its implementation in `lib/upstash`).
Practice drill: from memory, list the five settled operations in `recordClick`, then verify.

---

## Q2: What is `waitUntil` (or "background work after the response") and what are its failure semantics?

Round: JS/TS deep-dive / system design bridge
What it's really testing: event-loop + serverless runtime model beyond textbook.
Repo anchor: [apps/web/lib/middleware/link.ts#L553-L567](../../../apps/web/lib/middleware/link.ts#L553-L567); [lib/auth/workspace.ts#L272-L300](../../../apps/web/lib/auth/workspace.ts#L272-L300)
Junior answer sounds like: "it runs code after the response."
Mid-level answer adds: the response is sent first so p99 doesn't pay for analytics; failures don't affect the user and are only visible in logs; you can't use it for anything the response depends on.
Senior answer includes: it's a *durability* tradeoff — the platform keeps the instance alive but gives no delivery guarantee; anything that must happen should go to a queue (Dub uses QStash for those cases). Classify effects: best-effort (analytics) vs must-happen (billing).
Likely follow-ups: "What happens on instance crash?" "How is this different from `setImmediate`/not awaiting?" (un-awaited promises may be killed at response end in serverless; `waitUntil` explicitly extends the lifetime).
Practice drill: explain why `ev.waitUntil(recordClick(...))` appears *before* `return NextResponse.redirect(...)` yet runs after the user is redirected.

---

## Q3: How do you type a function that returns either a success or an error without exceptions?

Round: JS/TS deep-dive
What it's really testing: discriminated unions and narrowing.
Repo anchor: [apps/web/lib/api/links/process-link.ts#L44-L57](../../../apps/web/lib/api/links/process-link.ts#L44-L57)
Junior answer sounds like: `{data?: T, error?: string}` (both optional — the anti-pattern).
Mid-level answer adds: a union discriminated on `error: null` vs `error: string`, so `if (error != null)` narrows the other branch to the success type; points at `processLink`'s return type where `code?: never` in the success arm makes illegal states unrepresentable.
Senior answer includes: why this layer uses values (bulk composability) while the HTTP layer throws typed `DubApiError` — error strategy is a *per-layer* decision; also mentions the generic `<T extends Record<string, any>>` preserving the caller's extra fields through the pipeline.
Likely follow-ups: "What does `never` do there?"; "Result types vs exceptions in TS at scale?"
Practice drill: write the union from memory, then diff against the real one.

---

## Q4: Where does validation belong in a TypeScript API — and what do Zod schemas buy you over interfaces?

Round: JS/TS deep-dive
What it's really testing: runtime vs compile-time types.
Repo anchor: [apps/web/app/api/links/route.ts#L54-L56](../../../apps/web/app/api/links/route.ts#L54-L56); schema library at [lib/zod/schemas/](../../../apps/web/lib/zod/schemas/)
Junior answer sounds like: "TypeScript types check the request body."
Mid-level answer adds: interfaces are erased at runtime; network input needs runtime parsing; `z.infer` derives the static type from the runtime validator so they can't drift; Dub generates its OpenAPI docs from the same schemas.
Senior answer includes: parse-don't-validate at the boundary, trust inside; `parseAsync` variants exist because some validation needs I/O (e.g., `createLinkBodySchemaAsync`); Zod errors are mapped to a stable 422 contract in one place ([errors.ts#L100-L105](../../../apps/web/lib/api/errors.ts#L100-L105)).
Likely follow-ups: "Cost of parsing on hot paths?"; "How do you validate *outbound* data?" (Dub parses webhook payloads before sending — [links route L92](../../../apps/web/app/api/links/route.ts#L92)).
Practice drill: find one `.parse` and one `.parseAsync` call site and explain the difference.

---

## Q5: Explain the Node.js event loop. How does it shape how you write server code?

Round: JS/TS deep-dive
What it's really testing: can you connect the model to real decisions.
Repo anchor: conceptual — but ground it in the redirect path: [lib/middleware/link.ts](../../../apps/web/lib/middleware/link.ts) is one async function where every `await` is a yield point.
Junior answer sounds like: call stack, task queue, microtasks recital.
Mid-level answer adds: single thread means CPU work blocks *all* requests; hence Dub's hot path is pure I/O orchestration (cache reads, one DB query) and pushes enrichment (UA parsing is cheap, but Tinybird ingest, DB counters) into post-response work; serial `await`s add latency, so independent I/O is batched with `Promise.allSettled`.
Senior answer includes: microtask starvation, why timers/`setInterval` are unreliable under load, and serverless nuance — each instance handles limited concurrency so the loop matters less than cold starts and per-instance caches (the 5s LRU exists *because* instances are many and short-lived).
Likely follow-ups: "await in a loop vs Promise.all?" — point at the sequential `await`s in `LinkMiddleware` that are *intentionally* sequential (each depends on the previous).
Practice drill: in `LinkMiddleware`, mark which awaits could be parallelized and which are inherently serial.

---

## Q6: What's a closure? Show me a real bug or real use that isn't a counter example.

Round: JS/TS deep-dive
What it's really testing: whether you can connect closures to production code.
Repo anchor: [lib/middleware/link.ts#L112-L131](../../../apps/web/lib/middleware/link.ts#L112-L131) — the async IIFE inside `ev.waitUntil` closes over `linkData` and `isPartnerLink`.
Junior answer sounds like: "a function that remembers variables."
Mid-level answer adds: the `waitUntil(async () => {...})` pattern — the closure captures request-scoped data whose lifetime now *outlives the response*; and the higher-order `withWorkspace(handler, opts)` where `opts` is closed over for the life of the module.
Senior answer includes: closure-captured mutable variables across async boundaries are a stale-data hazard (capture-by-reference); module-level singletons like the LRU cache ([cache.ts#L18-L21](../../../apps/web/lib/api/links/cache.ts#L18-L21)) are closures over module scope that persist across requests in the same instance — both a feature (caching) and a leak risk.
Likely follow-ups: "Why can a module-level variable in a serverless function be both a cache and a bug?"
Practice drill: explain what `withWorkspace(handler, {requiredPermissions: ["links.write"]})` closes over and for how long.

---

## Q7: How do you handle errors consistently across a large API surface?

Round: JS/TS deep-dive / API design
What it's really testing: error architecture, not try/catch syntax.
Repo anchor: [apps/web/lib/api/errors.ts#L43-L130](../../../apps/web/lib/api/errors.ts#L43-L130)
Junior answer sounds like: "try/catch and return 500."
Mid-level answer adds: a typed error class with a machine-readable `code`, one mapper that handles Zod errors → 422, Prisma P2025 → 404, `DubApiError` → its mapped status, unknown → 500; wrappers guarantee every route passes through it ([workspace.ts#L502-L505](../../../apps/web/lib/auth/workspace.ts#L502-L505)).
Senior answer includes: error contract as public API (doc_url per code, mirrored into OpenAPI); logging on the error path *with request context but after clearing mis-attributed tenant state* ([workspace.ts#L362-L364](../../../apps/web/lib/auth/workspace.ts#L362-L364)); distinguishing operator errors (alert) from client errors (meter).
Likely follow-ups: "Where would you put Sentry/alerting in this?"
Practice drill: enumerate the branches of `handleApiError` from memory.

---

## Q8: TS generics — show me a generic that earns its complexity.

Round: JS/TS deep-dive
What it's really testing: generics as contracts, not syntax golf.
Repo anchor: [lib/api/links/process-link.ts#L24-L34](../../../apps/web/lib/api/links/process-link.ts#L24-L34) — `processLink<T extends Record<string, any>>({payload: NewLinkProps & T})`.
Junior answer sounds like: `function identity<T>(x: T): T`.
Mid-level answer adds: `processLink` is used by create, update, upsert, and bulk flows whose payloads carry *different extra fields*; the generic threads those through so callers get back their own type, validated — without the function knowing about them.
Senior answer includes: the tradeoff — `Record<string, any>` is a loose bound (any extra garbage flows through too); tighter alternative is a union of known payload types, at the cost of coupling the helper to every caller.
Likely follow-ups: "`unknown` vs `any`?"; "When do you reach for `satisfies`?"
Practice drill: write the signature from memory; explain what breaks if `T` is removed.

---

## Q9: What is an atomic increment and why does `UPDATE ... SET clicks = clicks + 1` matter vs read-then-write?

Round: JS/TS deep-dive / data
What it's really testing: concurrency basics in the language *and* the database.
Repo anchor: [lib/tinybird/record-click.ts#L194-L199](../../../apps/web/lib/tinybird/record-click.ts#L194-L199)
Junior answer sounds like: "it adds one to clicks."
Mid-level answer adds: read-modify-write in app code races under concurrency (two readers both see 41, both write 42); pushing the arithmetic into SQL makes the row lock do the work; JS being single-threaded does *not* save you because concurrency is across instances/requests.
Senior answer includes: this is also why the counter lives in MySQL rather than being summed from Tinybird on read; and the app-level analogue — Upstash `INCR`-based rate limiting — is the same primitive one layer up.
Likely follow-ups: "How would you do this in Prisma?" (`{ clicks: { increment: 1 } }` — and why they used raw SQL anyway: connection pooling on the hot path, per the comment at L195).
Practice drill: write both the racy and the safe version in pseudocode.

---

## Q10: How do module-level singletons behave in serverless, and where does this repo rely on that?

Round: JS/TS deep-dive / Node runtime
What it's really testing: module caching + serverless instance lifecycle.
Repo anchor: [lib/api/links/cache.ts#L18-L27](../../../apps/web/lib/api/links/cache.ts#L18-L27) (module-level LRU + `getCache()`); Prisma client singleton in [packages/prisma/client.ts](../../../packages/prisma/client.ts).
Junior answer sounds like: "modules are cached so it's created once."
Mid-level answer adds: once *per instance* — many instances exist, each with its own LRU, so it's a per-instance optimization with cross-instance inconsistency bounded by the 5s TTL; the Prisma client is a singleton to avoid connection exhaustion.
Senior answer includes: the comment at [cache.ts#L22-L26](../../../apps/web/lib/api/links/cache.ts#L22-L26) — under spike, *new* instances have cold LRUs, hence the Vercel-cache third tier; also warns about mutable module state as hidden cross-request coupling.
Likely follow-ups: "Why is a global in a Lambda dangerous?" "How do you test code with module singletons?"
Practice drill: predict what `[LRU Cache HIT]`/`[MISS]` logs look like during a traffic spike across 50 fresh instances.

---

## Q11: Explain cookies from Node's perspective: setting, scoping, and why the path matters.

Round: JS/TS deep-dive / web platform
What it's really testing: HTTP fundamentals through a JS lens.
Repo anchor: [lib/middleware/link.ts#L254-L279](../../../apps/web/lib/middleware/link.ts#L254-L279) and [lib/middleware/utils/create-response-with-cookies.ts](../../../apps/web/lib/middleware/utils/create-response-with-cookies.ts)
Junior answer sounds like: "cookies store data in the browser."
Mid-level answer adds: the `dub_id_<domain>_<key>` cookie is set *on the redirect response* with `path` scoped to the link key — so the click identity survives to power later conversion attribution, without leaking across links on the same domain.
Senior answer includes: cookie-on-302 semantics (browsers do store cookies from redirect responses), the privacy/consent angle, and the fallback chain when there's no cookie: Redis identity-hash cache, then a fresh `nanoid(16)`.
Likely follow-ups: "SameSite implications for a link that redirects cross-site?"
Practice drill: trace `clickId`'s three possible sources in order.

---

## Q12: What is punycode / unicode normalization doing in a URL system?

Round: JS/TS deep-dive / edge cases
What it's really testing: string handling maturity — most candidates have never thought about it.
Repo anchor: [lib/middleware/link.ts#L46-L52](../../../apps/web/lib/middleware/link.ts#L46-L52) (`punyEncode`, per-domain case sensitivity)
Junior answer sounds like: (blank).
Mid-level answer adds: user-visible keys can contain unicode; storage/lookup needs a canonical ASCII form; case-insensitivity is a *product* decision implemented as lowercase-at-the-edge — with per-domain opt-out ([lib/api/links/case-sensitivity.ts](../../../apps/web/lib/api/links/case-sensitivity.ts)), so normalization must match at write time and read time or links 404.
Senior answer includes: normalization mismatch between write and read paths is a classic incident class (works when created, 404s when clicked); homograph phishing as the security cousin of this topic.
Likely follow-ups: "Where else do you canonicalize?" (emails, usernames, search).
Practice drill: explain why `encodeKeyIfCaseSensitive` exists on the *write* path ([create-link.ts#L51-L54](../../../apps/web/lib/api/links/create-link.ts#L51-L54)).

---

## Q13: How does retry logic go wrong, and where should it live?

Round: JS/TS deep-dive / reliability
What it's really testing: idempotency awareness.
Repo anchor: `withPrismaRetry` at [lib/api/links/create-link.ts#L55](../../../apps/web/lib/api/links/create-link.ts#L55) (impl in [lib/api/utils/with-prisma-retry.ts](../../../apps/web/lib/api/utils/with-prisma-retry.ts)); `fetchWithRetry` for Tinybird ([record-click.ts#L180](../../../apps/web/lib/tinybird/record-click.ts#L180))
Junior answer sounds like: "wrap it in a loop with a counter."
Mid-level answer adds: retrying a **create** is only safe if a unique constraint makes the duplicate fail loudly (here `(domain,key)` and `shortLink` uniques act as the idempotency backstop); retries need jitter/backoff and a budget; retry reads freely, retry writes only when idempotent.
Senior answer includes: retry amplification during incidents (everything retrying at once), the difference between client-side retry (`fetchWithRetry`) and delegating retries to a queue (QStash for webhooks); the open question of whether the QStash publish itself is retried.
Likely follow-ups: "What makes an operation idempotent?" — define via keys, not vibes.
Practice drill: for each retry wrapper in the repo, state the idempotency argument that makes it safe (or find that there isn't one).

---

## Q14: `async` function that's never awaited — what actually happens?

Round: JS/TS deep-dive
What it's really testing: promise semantics, unhandled rejections, serverless gotchas.
Repo anchor: contrast the deliberate un-awaited-but-managed pattern `ev.waitUntil((async () => {...})())` at [lib/middleware/link.ts#L112-L131](../../../apps/web/lib/middleware/link.ts#L112-L131) with a bare un-awaited call.
Junior answer sounds like: "it still runs."
Mid-level answer adds: it runs, but rejections become unhandled-rejection events (process-level noise or crash depending on Node config), and in serverless the runtime may freeze/kill the instance right after the response — the work silently never finishes; `waitUntil` exists precisely to register the promise with the platform.
Senior answer includes: fire-and-forget still needs a `.catch` for *attribution* (see the named-operation error mapping at [record-click.ts#L233-L262](../../../apps/web/lib/tinybird/record-click.ts#L233-L262)); lint rules (`no-floating-promises`) as the structural fix.
Likely follow-ups: "How would you find floating promises in a big codebase?"
Practice drill: find one `.catch(() => undefined)` in `lib/middleware/link.ts` (hint: [#L263-L265](../../../apps/web/lib/middleware/link.ts#L263-L265)) and argue whether swallowing there is correct.
