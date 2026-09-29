# Security Checklist

Threat classes mapped to this repo's actual defenses (or gaps), then a pre-merge checklist. "Investigate" = raise, don't assert.

| Threat | This repo's defense | Anchor | Notes |
| --- | --- | --- | --- |
| Broken authn | hashed tokens, expiry checks, session fallback | [workspace.ts#L175-L235](../../../apps/web/lib/auth/workspace.ts#L175-L235) | investigate: token-cache invalidation on revoke |
| IDOR | tenant-scoped lookups, composite uniques, 404-not-403 | [get-link-or-throw.ts](../../../apps/web/lib/api/links/get-link-or-throw.ts); [workspace.ts#L386-L389](../../../apps/web/lib/auth/workspace.ts#L386-L389) | convention, not compiler-enforced — audit every new `find*` |
| Privilege escalation | role→permission table; token scopes **intersected** with role | [permissions.ts](../../../apps/web/lib/api/rbac/permissions.ts); [workspace.ts#L413-L418](../../../apps/web/lib/auth/workspace.ts#L413-L418) | machine users auto-owner ([#L404-L408](../../../apps/web/lib/auth/workspace.ts#L404-L408)) — review any integration granting machine users |
| Input validation | Zod at every route boundary; URL parsing/normalization | [lib/zod/schemas](../../../apps/web/lib/zod/schemas/); [process-link.ts#L85-L93](../../../apps/web/lib/api/links/process-link.ts#L85-L93) | remember middleware inputs (domain/key) are validated by *parsing*, not schemas |
| SQL injection | Prisma parameterization; raw SQL uses `?` placeholders | [record-click.ts#L196-L199](../../../apps/web/lib/tinybird/record-click.ts#L196-L199), [#L324-L341](../../../apps/web/lib/tinybird/record-click.ts#L324-L341) | any new `conn.execute` with template literals = blocking review finding |
| Open redirect / abuse | this product *is* a redirector — defenses are blacklists + banning | `isBlacklistedDomain` ([process-link.ts#L1](../../../apps/web/lib/api/links/process-link.ts#L1)); banned rewrite [link.ts#L212-L221](../../../apps/web/lib/middleware/link.ts#L212-L221); `.php` key rejection [#L73-L83](../../../apps/web/lib/middleware/link.ts#L73-L83); anonymous links self-delete in 30 min | the fraud/ dir ([lib/api/fraud](../../../apps/web/lib/api/fraud)) exists for the partner side |
| SSRF | metatag/proxy fetching of user URLs is the risk surface | [app/api/links/metatags](../../../apps/web/app/api/links/metatags) | investigate: does the fetcher block private IP ranges / redirects-to-internal? A classic place to check in any link tool |
| XSS | React escaping by default; OG-proxy pages render user titles/images | [app/[domain]/[key]/proxy pages] (under `app/`) | investigate `dangerouslySetInnerHTML` usage repo-wide; tiptap rich text is a second surface |
| CSRF | session APIs rely on NextAuth cookie handling; server actions have Next's origin checks; Bearer-token API is CSRF-immune by design | [lib/auth/options.ts](../../../apps/web/lib/auth/options.ts) | investigate SameSite settings on session cookie |
| Webhook (outbound) authenticity | HMAC `Dub-Signature` per-webhook secret | [qstash.ts#L67](../../../apps/web/lib/webhook/qstash.ts#L67), [lib/webhook/signature.ts](../../../apps/web/lib/webhook/signature.ts) | consumers must verify; duplicates possible (no dedup id) |
| Webhook/cron (inbound) authenticity | QStash signature verification | [verify-qstash.ts](../../../apps/web/lib/cron/verify-qstash.ts) | **skipped when `VERCEL !== "1"`** (L18-L21) — env-conditional security; fine on-platform, a hole if self-hosted |
| Secrets handling | tokens hashed; publishable vs secret key split ([lib/auth/publishable-key.ts](../../../apps/web/lib/auth/publishable-key.ts)); env vars for service creds | [hash-token.ts](../../../apps/web/lib/auth/hash-token.ts) | never log bodies without redaction (kata 7) |
| PII / privacy | EU IP redaction; `workspacePreferences` hidden from API | [record-click.ts#L143-L145](../../../apps/web/lib/tinybird/record-click.ts#L143-L145); [workspace.ts#L355](../../../apps/web/lib/auth/workspace.ts#L355) | investigate identityHash vs GDPR |
| Rate limiting / abuse | plan-based API limits; per-second analytics limits; anonymous 10/day/IP; click dedup | [workspace.ts#L237-L265](../../../apps/web/lib/auth/workspace.ts#L237-L265); [links route L58-L69](../../../apps/web/app/api/links/route.ts#L58-L69) | limits surfaced in headers — good client behavior enabled |
| Dependency risk | pnpm lockfile; `resolutions` pin | [pnpm-lock.yaml](../../../pnpm-lock.yaml) | no automated audit visible in-repo (CI config not fully surveyed — investigate `.github/`) |
| Uploads | images go to R2 with size constraints via the storage helper | [lib/storage.ts](../../../apps/web/lib/storage.ts); base64 image schemas in [lib/zod/schemas/images.ts](../../../apps/web/lib/zod/schemas/images.ts) | investigate content-type validation and SVG handling (script-in-SVG) |

## Pre-merge security checklist (use on every PR you write or review here)

- [ ] Every new `prisma.*.find/update/delete` carries the tenant scope or goes through a scoped helper.
- [ ] New route has explicit `requiredPermissions` (absence = any member can call it — deliberate?).
- [ ] New inputs are Zod-parsed; new *outputs* don't leak fields (diff the response schema).
- [ ] Raw SQL uses placeholders; no string-built queries.
- [ ] Any fetch of a user-supplied URL considers SSRF (IP ranges, redirects, timeouts).
- [ ] No secrets/PII in logs, error messages, or webhook payloads.
- [ ] Background/cron endpoints verify their caller (QStash signature) — and note the local-skip.
- [ ] Rate limiting story for any new unauthenticated surface.
- [ ] Folder-ACL check present if the resource can live in a folder.
- [ ] Failure mode reviewed: does an error path fail *open* (skip a check) or *closed*?

Drill: run the checklist against kata 5 and kata 8 in [review katas](../04-code-reading-gym/04-review-katas.md) — each violates at least two rows. Then pick one "investigate" cell above, spend 30 minutes resolving it in the code, and record the answer in the [risk register](../09-reference/risk-register.md).

Interview angle: security rounds for fullstack mids are usually "walk me through how you'd secure this endpoint" — answer with the ladder from [03/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) plus two concrete war-story details from this table (hashed tokens + scope intersection, or the env-conditional signature skip as a "what I'd flag" example).
