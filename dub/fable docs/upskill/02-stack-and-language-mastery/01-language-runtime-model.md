# Language & Runtime Model (JS/Node in serverless)

## Mental model from first principles

One thread per instance runs your JS. Every `await` yields; between yields nothing else in *this instance* runs. Concurrency you must reason about is therefore **across requests and instances**, not within a function. In serverless, instances are created on demand (cold start), reused (warm), and killed unpredictably — so module scope is "per-instance memory of uncertain lifetime."

## Where the repo uses this well

- **Post-response work registered with the platform**: `ev.waitUntil(...)` / `waitUntil(...)` instead of floating promises — the platform keeps the instance alive ([link.ts#L553-L567](../../../apps/web/lib/middleware/link.ts#L553-L567), [workspace.ts#L486-L499](../../../apps/web/lib/auth/workspace.ts#L486-L499)). A bare un-awaited promise might never finish after the response is sent.
- **Independent I/O batched**: [record-click.ts#L179-L230](../../../apps/web/lib/tinybird/record-click.ts#L179-L230) settles five writes concurrently rather than awaiting serially — the difference between ~max(latencies) and ~sum(latencies).
- **Module-scope state used deliberately**: the 10k-entry LRU ([cache.ts#L18-L21](../../../apps/web/lib/api/links/cache.ts#L18-L21)) exploits instance reuse, with a 5s TTL because instance-local caches can't be invalidated remotely; the comment at [#L22-L26](../../../apps/web/lib/api/links/cache.ts#L22-L26) shows they know fresh instances arrive cold.
- **Atomicity delegated to the database**: `clicks = clicks + 1` in SQL ([record-click.ts#L196-L199](../../../apps/web/lib/tinybird/record-click.ts#L196-L199)) because app-level read-modify-write races across instances.

## Sharp edges in this repo

- `.catch(() => undefined)` on the recordClickCache read ([link.ts#L263-L265](../../../apps/web/lib/middleware/link.ts#L263-L265)) — correct here (best-effort), but the pattern copied to a load-bearing call would swallow real failures.
- Conditional entries in the allSettled array (`workspaceId && publish...`) evaluate to `false` when the guard fails — allSettled treats non-promises as fulfilled, so result indexes shift meaning. See ticket 2.
- Env-dependent behavior: `process.env.VERCEL === "1"` switches geo/IP handling ([record-click.ts#L118-L129](../../../apps/web/lib/tinybird/record-click.ts#L118-L129)) — local runs literally exercise different code.

## Pitfall checklist

- [ ] Did I `await` (or `waitUntil`) every promise? (`no-floating-promises` mindset)
- [ ] Is any of this work CPU-heavy enough to block the loop? (JSON.parse of huge bodies, sync crypto, regex backtracking)
- [ ] Am I reading-then-writing shared state (DB row, Redis key) without an atomic primitive?
- [ ] Does module-level state assume a single instance? What's the staleness bound?
- [ ] Do independent awaits run serially for no reason?

## Drills

1. In [LinkMiddleware](../../../apps/web/lib/middleware/link.ts), mark every `await` as serial-required or parallelizable; justify each.
2. Predict console output ordering of the `[LRU Cache ...]` logs for two concurrent requests to the same key on one warm instance, then on two cold instances.
3. Find one place a `waitUntil` failure would eventually surface to a user (hint: usage counters gate quota checks) and describe the delay between failure and symptom.

## Interview angle

- "all vs allSettled vs race" → [08/01 Q1](../08-interview-prep/01-js-ts-node-deep-dive.md)
- "background work after response" → [08/01 Q2](../08-interview-prep/01-js-ts-node-deep-dive.md)
- "event loop, but connected to real decisions" → [08/01 Q5](../08-interview-prep/01-js-ts-node-deep-dive.md)
- "module singletons in serverless" → [08/01 Q10](../08-interview-prep/01-js-ts-node-deep-dive.md)
- "atomic increments / race conditions" → [08/01 Q9](../08-interview-prep/01-js-ts-node-deep-dive.md)
