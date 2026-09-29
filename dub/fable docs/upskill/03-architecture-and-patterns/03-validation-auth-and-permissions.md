# Validation, Auth, and Permissions

## Vocabulary first

**Authentication** (authn): who are you. **Authorization** (authz): what may you do. **Isolation**: tenant A cannot observe tenant B even by misbehaving. **IDOR**: authz bug where knowing an id grants access. Interviews probe whether you keep these distinct.

## The validation ladder (request lifecycle)

| Layer | What it rejects | Where |
| --- | --- | --- |
| 1. Shape | wrong types, missing fields, bad URLs | Zod at route top ([links route L54-L56](../../../apps/web/app/api/links/route.ts#L54-L56)); schema library [lib/zod/schemas](../../../apps/web/lib/zod/schemas/) |
| 2. Authn | bad/expired credentials | [workspace.ts#L105-L235](../../../apps/web/lib/auth/workspace.ts#L105-L235) — session or hashed Bearer token |
| 3. Rate | abuse | plan-based limits [#L237-L265](../../../apps/web/lib/auth/workspace.ts#L237-L265); anonymous 10/day/IP ([links route L58-L69](../../../apps/web/app/api/links/route.ts#L58-L69)) |
| 4. Tenancy | non-members | membership join fetch [#L342-L402](../../../apps/web/lib/auth/workspace.ts#L342-L402); 404 not 403 (anti-enumeration) |
| 5. Permission | role/scope insufficient | role→permissions ∩ token scopes [#L404-L439](../../../apps/web/lib/auth/workspace.ts#L404-L439); vocabulary in [rbac/permissions.ts](../../../apps/web/lib/api/rbac/permissions.ts) |
| 6. Plan/flags | feature not purchased/enabled | [#L441-L473](../../../apps/web/lib/auth/workspace.ts#L441-L473); deeper gates in domain ([plan-features-check.ts](../../../apps/web/lib/api/links/plan-features-check.ts)) |
| 7. Business | invalid state transitions, blacklists, quotas | [process-link.ts](../../../apps/web/lib/api/links/process-link.ts) (URL rules, blacklisted domains), `throwIfLinksUsageExceeded` |
| 8. Resource scope | foreign ids | scoped lookups: [get-link-or-throw.ts](../../../apps/web/lib/api/links/get-link-or-throw.ts) takes `workspaceId`; program check [analytics route L52-L60](../../../apps/web/app/api/analytics/route.ts#L52-L60) |
| 9. Sub-tenant ACL | folder restrictions | [lib/folder/permissions.ts](../../../apps/web/lib/folder/permissions.ts) via `verifyFolderAccess` ([analytics route L94-L101](../../../apps/web/app/api/analytics/route.ts#L94-L101)) |

Memorize the shape, not the numbers: **shape → identity → abuse → tenant → permission → money → business → resource → sub-resource**. Any multi-tenant SaaS has some version of this ladder; being able to *recite and locate* it is a hiring signal.

## Isolation mechanics worth quoting

- Tokens are stored **hashed** ([hash-token.ts](../../../apps/web/lib/auth/hash-token.ts)); presented keys are hashed then matched — DB leak ≠ credential leak.
- Restricted-token scopes are **intersected** with the user's role permissions ([workspace.ts#L413-L418](../../../apps/web/lib/auth/workspace.ts#L413-L418)) — a token can narrow, never widen. That's the invariant to state in interviews.
- Resource lookups take the tenant id as a *parameter* (structural IDOR defense) rather than trusting a route-level check someone must remember.
- Cross-tenant probes get `not_found`, not `forbidden` — existence itself is protected ([#L386-L389](../../../apps/web/lib/auth/workspace.ts#L386-L389)).
- Public-surface gating is *state-based*: password/expiry/ban on the redirect ([link.ts#L175-L252](../../../apps/web/lib/middleware/link.ts#L175-L252)) — authorization without authentication.

## What a junior misses vs what a senior checks

Junior misses: that Zod passing says nothing about *authorization*; that the folder layer exists at all (intra-workspace ACLs); that server actions ([safe-action.ts](../../../apps/web/lib/actions/safe-action.ts)) are a **second, parallel auth implementation** that must be audited separately; that the anonymous carve-out ([workspace.ts#L134-L146](../../../apps/web/lib/auth/workspace.ts#L134-L146)) exists.

Senior checks, in order: every new endpoint's `requiredPermissions` option (absent = any member); every `prisma.<resource>.find*` in the diff for tenant scoping; whether new resources join the folder-ACL system or bypass it; token-cache invalidation on revocation (open question — investigate [token-cache.ts](../../../apps/web/lib/auth/token-cache.ts)); machine-user auto-owner behavior ([#L404-L408](../../../apps/web/lib/auth/workspace.ts#L404-L408)) whenever integrations grow capabilities.

Interview angle: → [08/03 Q4, Q5](../08-interview-prep/03-api-and-data-modeling-questions.md) (authz/IDOR), [08/04 step 5](../08-interview-prep/04-system-design-from-this-repo.md) (multi-tenancy), and the security round in [05/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).

Drill: write the "ladder table" above for `PATCH /api/links/[linkId]` from scratch by reading [app/api/links/[linkId]/route.ts](../../../apps/web/app/api/links/%5BlinkId%5D/route.ts) — then verify each rung exists. Self-grade — Basic: found Zod + withWorkspace. Solid: found the scoped lookup and permission option. Strong: checked folder access and identified any rung that's *missing* relative to the POST path.
