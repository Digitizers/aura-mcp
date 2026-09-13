# `aura__*` tool reference

All tools are served by the Aura fleet gateway (`/api/mcp/fleet`) and visible only to a
`canManage` (management) token. Every tool is scoped to the token's **organization +
client**, and honours the token's **pack** (Site-Group) restriction if it has one. Secrets
(encrypted provider credentials) are **never** returned.

List rows default to **50**, max **200** (`limit` argument, clamped server-side).

---

## Reads (always available on a management token)

### `aura__list_pending_approvals`
Agent actions awaiting human approval in scope (the governed writes the fleet queued).
- **Args:** `limit?` (int)
- **Returns:** `{ approvals: [{ id, tool, source, resourceId, runId, actorType, params, requestedByUserId, createdAt }] }`
- Use to see what's waiting, then drill in with `aura__get_action` or act with
  `aura__reject_action` / `aura__approve_action`.

### `aura__get_action`
One action's full detail — status, params, result, approval/rejection metadata.
- **Args:** `actionId` (string, **required**)
- **Returns:** `{ action: { id, tool, source, resourceId, runId, actorType, status, params, result, success, errorMessage, requestedByUserId, approvedByUserId, approvedAt, rejectedByUserId, rejectedAt, rejectionReason, startedAt, finishedAt, createdAt, updatedAt } }`
- An out-of-scope / out-of-pack id resolves to `null` (returned as an error), never a leak.

### `aura__list_snapshots`
Page **and file** snapshots captured before governed writes — the rollback points, merged
newest-first by `createdAt` and capped once at `limit` (not `limit` per table).
- **Args:** `resourceId?` (string), `postId?` (int), `limit?` (int)
- **Returns:** `{ snapshots: [Page | File] }` where:

  - a **page** row is `{ type: "page", id, resourceId, postId, tool, agentActionId, runId, restoredAt, createdAt }`
  - a **file** row is `{ type: "file", id, resourceId, agentActionId, runId, op, path, fenced, restoredAt, createdAt }`
    — no `postId` and no `tool`; `op` is the write that was captured, `path` is the file's
    path, `fenced` says whether Aura holds a rollback record the site can act on
- Supplying `postId` returns **page rows only**.
- A pack-scoped token passing a `resourceId` outside its pack matches nothing (the request
  can't widen the pack).

### `aura__list_connections`
Provider connections (Cloudflare / Bunny / Cloudways / Hostinger / Vultr / xCloud) with
validation status. **Never** returns credentials.
- **Args:** `limit?` (int)
- **Returns:** `{ connections: [{ id, provider, displayName, status, lastValidatedAt, validationFailureCount, createdAt }] }`

### `aura__list_runs`
Recent runs — actions grouped by `runId`, each with its action count and latest activity.
Ordered by last activity (an approved/started/finished action bumps a run up).
- **Args:** `limit?` (int)
- **Returns:** `{ runs: [{ runId, actionCount, lastActivityAt }] }`
- Feed a `runId` to `aura__rollback_run` to unwind a whole run.

### `aura__client_summary`
One-shot situational snapshot of the token's client scope.
- **Args:** none
- **Returns:** `{ summary: { resourceCount, connectionCount, pendingApprovals, snapshotCount } }`
- `snapshotCount` counts page and file rollback points together.

---

## Writes (governed — read [safety.md](safety.md) first)

### `aura__reject_action` — safe write
Reject a pending action so it never runs. It can only **deny** a queued action, never
execute one, so it rides the default allowlist (no explicit opt-in needed).
- **Args:** `actionId` (string, **required**), `reason?` (string, ≤500 chars)
- **Returns:** `{ rejected: true, actionId, status }` — or `{ rejected: false, code }` on error
  (e.g. the action was already decided).

### `aura__restore_snapshot` — high-risk revert
Roll a **page or file** back to a captured snapshot (undo a design write, or a file an agent
created/overwrote). Idempotent — an already-restored snapshot is a no-op. Recorded as a
self-approved action for the audit trail.
- **Requires:** a **client-wide** management token (a pack-scoped token is refused), **and**
  `aura__restore_snapshot` named in the token's `allowedTools`. Restoring a **file** id also
  requires the separate `aura__restore_snapshot:file` capability in `allowedTools` — a plain
  `aura__restore_snapshot` grant does not include it.
- **Args:** `snapshotId` (string, **required**)
- **Returns (page):** `{ ok, code, actionId, postId }`
- **Returns (file):** `{ ok, code, actionId }` — no `postId`
- `actionId` is present only when **this call** actually settled the restore. If another claim
  took the row over mid-flight, the call still answers `ok`, but without an `actionId` — the
  action it created is no longer the one that owns the outcome.
- **Missing the file capability:** `{ ok: false, code: "FILE_RESTORE_CAPABILITY_REQUIRED" }` —
  and only when `snapshotId` genuinely resolves to a **file** snapshot in the token's own
  scope. An id matching neither table still answers the file table's `NOT_FOUND` — a
  capability refusal for something that doesn't exist would be inaccurate, not merely vague.
  This code never doubles as "unknown id."

### `aura__rollback_run` — high-risk revert
Roll back a whole run — replays every snapshot the run produced (infra + page + file)
newest-first. A partial failure never aborts the rest.
- **Requires:** a **client-wide** token (a run can span resources outside a pack) **and**
  `aura__rollback_run` in `allowedTools`. A **file** leg additionally needs
  `aura__restore_snapshot:file` — without it, that leg is reported `not_attempted` rather than
  run, while the rest of the run still executes.
- **Args:** `runId` (string, **required**)
- **Returns:** `{ ok, restored, failed, results: [{ ...perSnapshot, status }] }` (`ok` is
  false if any snapshot failed).
- `results[].status` is one of **`restored`**, **`failed`**, or **`not_attempted`**:
  - `failed` — the site was asked and refused.
  - `not_attempted` — nothing was sent. A file leg the token isn't entitled to restore is
    reported this way, with a reason string beginning `FILE_CAPABILITY_REQUIRED` (note: a
    different string from `aura__restore_snapshot`'s `FILE_RESTORE_CAPABILITY_REQUIRED` code
    above — the two are not the same value, don't conflate them). `not_attempted` also covers
    legs the run couldn't start before its request budget ran out.

### `aura__approve_action` — most privileged (approve AND run)
Approve a pending action and **execute** the queued write. The tool + params come from the
recorded action — you only allow what was already requested.
- **Requires ALL of:**
  1. `canManage` token (as with every `aura__*` tool),
  2. `aura__approve_action` named in the token's `allowedTools` (explicit opt-in),
  3. the **organization** opted into machine-approve (off by default — otherwise approvals
     stay human-tap in the Aura UI),
  4. a token with a **known owner** (its creator) — an ownerless token is refused,
  5. **not** self-approval — a token may not approve an action it requested.
- **Args:** `actionId` (string, **required**)
- **Returns:** `{ ok, actionId, status, success }` — or `{ ok: false, code }` with a code
  like `ORG_OPT_IN_REQUIRED`, `ACTOR_REQUIRED`, or the self-approval refusal.

---

## Error codes you'll see

| Code | Meaning |
|---|---|
| `ORG_OPT_IN_REQUIRED` | Machine-approve isn't enabled for the org — approve in the Aura UI. |
| `ACTOR_REQUIRED` | The management token has no known owner; re-mint under an active user. |
| `PACK_SCOPED_TOKEN` | A revert (restore/rollback) needs a client-wide token, not a pack one. |
| `FILE_RESTORE_CAPABILITY_REQUIRED` | `aura__restore_snapshot` on a **file** id, from a token that lacks `aura__restore_snapshot:file`. Fires only when the id genuinely resolves to a file snapshot in scope — an id matching neither table is `NOT_FOUND` instead, never this code. |
| (self-approval refusal) | The token requested the action it's trying to approve. |
