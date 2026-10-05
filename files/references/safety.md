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
- **`aura__reject_action`** and **`aura__reject_run`** — can only **deny** queued actions,
  never execute one. They ride the default (empty) allowlist. A token with an **explicit**
  `allowedTools` list gets only what the list names, so a read-only/reject-only token must name
  both.

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

## Which sites a fleet call runs on

A site tool called through the gateway takes `_sites` — an array of resource ids, or `"all"`
(see [tools.md](tools.md#naming-the-sites-a-call-runs-on-_sites); ids come from
`aura__list_sites`). **Name the sites.** Without `_sites`:

- a **read** runs at once on **every** connected site the token can see;
- anything the gateway does not recognise as a read is **refused** — it used to queue one
  approval per site, and now nothing is queued until the call says which sites, or `"all"`.

"Read" is decided by the tool's name: `run_wp_cli` is not a read whatever the command, and a
builder tool whose name does not start with a read verb (`get-`, `list-`, …) is not one unless
Aura declares it. With `_sites: "all"` such a tool queues **one approval per site** in scope.

So for a check on one site, pass that site's id — do not reach for `"all"` to see what a tool
does. If a call queued by mistake, its result carries a `runId` — clear it with
`aura__reject_run`, and **do not stop at the first answer**. One call rejects at most 200
actions and stops before the gateway's deadline; and a run that was still being created when
you called can gain actions afterwards. Repeat until the answer has `remaining: 0`,
`more: false`, an empty `notRejected` (or only entries you have looked at) **and**
`seal: "sealed"`. On `seal: "unsealed"`, call `aura__reject_run` again;
"nothing pending" on an unsealed run is not the end of it. A run whose creation was cut off is
never sealed: once the call that created it has returned, call `aura__reject_run` for that
`runId` one more time, and take its own `NOTHING_PENDING` as the end. Do not use the pending
list as the proof — it is capped at 200 rows and has no `runId` filter, so a run's actions can
be waiting outside the page it returned.

Tools seen to run at once (2026-10-05): `check_health`, `get_site_context`,
`elementor__elementor-mcp-server-info`. `elementor__elementor-mcp-detect-elementor-version` is
declared a read since `Digitizers/Aura#669`; before that it queued.
