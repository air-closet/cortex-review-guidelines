# Internal Member Identifier Guidelines

Rules for handling identifiers in any MCP tool, API, or batch that takes an internal
member (employee) as an argument. Introduced in issue [#665](https://github.com/air-closet/cortex/issues/665).

## Rules

Any process that accepts an internal member as an identifier must satisfy the
following two requirements.

1. **Accept either an email or a nickname**
   - Callers can identify the user with whichever piece of information they
     already have on hand
   - Designs that accept only email or only nickname are forbidden
2. **Resolve member identity against Google Workspace sync data**
   - Match the supplied email or nickname against the Google Workspace directory
     (in cortex, the Firestore `(default)/users` collection is the source of
     truth)
   - Error out if the user does not exist, has left the company, or is suspended
   - Determine `(email, nickname, displayName)` from the lookup result and use
     only the master values from that point forward

## Why

If callers can bring in arbitrary strings as identity, that identity will drift
from the user's real identity. For example, if a user whose true nickname is
`kei` can be registered as `k.terayama`, fixing the inconsistency later requires
a destructive `remove_member` → `add_member` cycle, and the system cannot keep up
with departures or name changes. Pinning the single source of truth for identity
to Google Workspace eliminates inconsistent naming, mis-registration, and broken
history at the root.

## Standard implementation in cortex

`resolveMemberIdentity(identifier)` in
`apps/mcp/git-server/src/sandbox/member-directory.ts` is the standard cortex
implementation. It joins against the Firestore `(default)/users` collection,
which `apps/pipeline/workspace/` syncs daily from Google Workspace.

### Expected return value

```ts
type ResolvedMemberIdentity = {
  email: string;       // 小文字化済み・マスタ値
  nickname: string;    // マスタ値（HR スプレッドシートまたは GWS 由来）
  displayName: string; // googleWorkspace.name.fullName 由来。空なら nickname にフォールバック
};
```

### Error conditions

- The identifier is empty
- No member matches in the `users` collection
- `googleWorkspace.archived` or `googleWorkspace.suspended` is set
- `status` is not `active`
- nickname is not registered (prompt the user to update the HR spreadsheet)

### Identifier detection

Treat any identifier containing `@` (i.e. `identifier.includes('@')`) as an
email. Otherwise treat it as a nickname: first look up `nickname` (lowercase
canonical form), then fall back to `nickName` (camelCase, for backward
compatibility) if it is empty.

## Parameter contract that tools must follow

```text
add_member(project_id, identifier, role)   # email | nickname のどちらか
remove_member(project_id, identifier)
update_member(project_id, identifier, ...) # role 変更時も nickname をマスタで再同期
```

- Designs that take legacy `user_email` and `nickname` as separate parameters
  are forbidden
- Do not let callers bring in their own nickname and overwrite with it; only
  the master value may be persisted
- For tools that must handle cases where the `users` lookup is impossible —
  e.g. removing members who have already left — first match the identifier
  against the tool's own member list (such as `project_members`), then fall
  back to `resolveMemberIdentity` (see `resolveProjectMemberEmail` in
  `apps/mcp/git-server/src/sandbox/project-tools.ts` for the reference pattern)

## Scope of application

- `sandbox_project_add_member` / `update_member` / `remove_member` in the
  `sandbox MCP` (applied in this PR)
- All other existing tools should be reviewed against this guideline over time
  (every MCP tool, API, or batch that takes an internal member as an
  identifier)
