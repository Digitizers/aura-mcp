# Governance & safety model

Aura's whole pitch is **human-in-the-loop governance**: every mutating action is queued and
approved, snapshotted, and audited. Driving that control plane over MCP must not weaken it.
These rules are enforced **server-side in the gateway** — they are not suggestions you can
route around.

## The meta-approval problem

An agent approving its **own** pending action over MCP would defeat the human-in-the-loop.
So `aura__approve_action` (the only tool that **executes** a queued write) is gated hard:

1. **`canManage`** — like every `aura__*` tool.
2. **Explicit allow-list** — the token must name `aura__approve_action` in its `allowedTools`.
   It is deliberately excluded from the "empty allowedTools = unrestricted" default, so an
   existing management token never silently gains execute-power when this tool ships.
3. **Org opt-in** — the organization must have enabled machine-approve (off by default). Until
   then, approvals stay **human-tap in the Aura UI**; the tool returns `ORG_OPT_IN_REQUIRED`.
4. **Known owner** — the token must have a live creator (its recorded actor). An ownerless
   token is refused (`ACTOR_REQUIRED`) rather than approving unattributably.
5. **No self-approval** — a token may not approve an action it (its creator) requested. The
   gateway refuses it.
6. **Distinct audit** — machine-approvals are flagged as approved-via-MCP in the trail.

**Default posture:** surface pending actions and comment on them; let a human tap Approve.
Only reach for `aura__approve_action` when the user has clearly set up and asked for
machine-approve.

## Reverts are client-wide only

`aura__restore_snapshot` and `aura__rollback_run` unwind changes that can span multiple
resources. A pack-scoped token can't safely bound that to a pack subset, so the gateway
**refuses** a pack-scoped token (`PACK_SCOPED_TOKEN`) rather than partially reverting. Use a
client-wide management token for reverts. Both also require explicit allow-listing.

## File restores are a separate, bigger consent

`aura__restore_snapshot` and `aura__rollback_run` cover two kinds of rollback point:
**page** snapshots (roll a page's content back) and **file** snapshots (delete a file the
agent created, or put back the content it overwrote). Both tables use cuid-shaped ids, so
`restore_snapshot` looks a given id up in the page table first, then the file table — but a
token that can restore pages cannot restore files just because it holds
`aura__restore_snapshot`.

Deleting or overwriting a file on the site is a bigger act than reverting a page's content, so
it needs its own, explicit opt-in: `aura__restore_snapshot:file` named in the token's
`allowedTools`, alongside `aura__restore_snapshot`. An existing token that was granted
`aura__restore_snapshot` before file restores existed consented to rolling back a *page* — the
grant is never widened silently to include deleting files.

Without that capability:

- `aura__restore_snapshot` on a **page** id is unaffected — it works exactly as before.
- `aura__restore_snapshot` on a **file** id is refused: `{ ok: false, code:
  "FILE_RESTORE_CAPABILITY_REQUIRED" }` — but only when the id genuinely resolves to a file
  snapshot in the token's scope; an id matching neither table still answers `NOT_FOUND`.
- `aura__rollback_run` still executes its resource and page legs. Each **file** leg is
  reported `not_attempted` with reason `FILE_CAPABILITY_REQUIRED` — never silently skipped,
  and never a whole-run refusal.

**Fix:** re-issue the token with `aura__restore_snapshot:file` added to `allowedTools`. It is
gated exactly like the other high-risk writes — client-wide token, explicit allow-listing —
nothing about the ordinary revert rules above changes for it.

## What's safe by default

- **All reads** (`list_*`, `get_action`, `client_summary`) — zero mutation risk.
- **`aura__reject_action`** — can only **deny** a queued action, never execute one. It rides
  the default allowlist.

## Scope is always enforced

Every tool is bound to the token's **organization + client**, and honours its **pack** if set.
An out-of-scope id resolves to nothing — never a cross-tenant leak. Encrypted credentials are
never returned by any tool.

## Token hygiene

- Mint management tokens sparingly and short-lived; they are your highest-privilege tokens.
- Give a token only the allowed-tools it needs — don't add `approve_action` to a token that
  only needs to reject.
- Store the `aura_` token in a secret manager or environment variable (`AURA_MCP_TOKEN`);
  never commit it. Tracked config files carry only `${AURA_MCP_TOKEN:-}` placeholders, and
  Cursor configs interpolate the env var (`${env:AURA_MCP_TOKEN}`); the one client that must
  embed the token inline (Claude Desktop) keeps its config out of version control.
