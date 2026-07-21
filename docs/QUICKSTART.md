# Quickstart — connect Claude to Aura in 5 minutes

**You need:** an Aura account ([app.my-aura.app](https://app.my-aura.app)) and an MCP client
(Claude Code, Claude Desktop, or Cursor).

## 1. Mint a management token (1 min)
Aura → **Fleet** → **Agent Tokens** → new token → check **"allow manage"** → **Copy** the
`aura_…` token (shown once).

- Read-only / reject-only? **Explicitly allow-list** the read tools + `aura__reject_action` —
  do NOT leave allowed-tools empty (empty = *unrestricted*, i.e. mutating fleet access too). See
  [connect.md](../files/references/connect.md) step 5.
- Want reverts or machine-approve? See step 4.

## 2. Supply the token (1 min)
Claude Code — export the token as an environment variable; never write it into a tracked file:
```bash
export AURA_MCP_TOKEN=aura_YOUR_MANAGEMENT_TOKEN
```
In **this repo's clone** (and claude.ai cloud environments that set the variable) the committed
placeholder-only `.mcp.json` picks it up automatically. In **your own project**, commit the
same placeholder config (`"Authorization": "Bearer ${AURA_MCP_TOKEN:-}"`) and keep the token
in the environment — see [`../files/references/connect.md`](../files/references/connect.md)
for the full config plus the Desktop / Cursor variants. Cursor interpolates the env var too
(`${env:AURA_MCP_TOKEN}`); only Claude Desktop needs the token inline — keep its config file
out of version control.

## 3. Verify (30 s)
Ask Claude: *"run aura__client_summary"*. Counts back → connected. No `aura__*` tools →
the token isn't `canManage`; re-mint with "allow manage".

## 4. (Optional) enable the high-risk writes
- **Reverts** (`aura__restore_snapshot`, `aura__rollback_run`): use a **client-wide** token
  (not pack-scoped) and add those tool names to the token's allowed-tools.
- **Machine-approve** (`aura__approve_action`): add it to allowed-tools **and** have an org
  admin enable machine-approve in Aura settings. Otherwise approvals stay human-tap in the UI
  (which is the safe default — the skill will surface + comment on pending actions instead).

## 5. Try it
- *"What's pending approval?"* → `aura__list_pending_approvals`
- *"Give me a summary of this client"* → `aura__client_summary`
- *"Reject action X, it's wrong"* → `aura__reject_action`
- *"Roll back run Y"* → `aura__rollback_run` (client-wide token)
