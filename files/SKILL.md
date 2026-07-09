---
name: aura-mcp
version: 0.1.1
license: MIT
description: Drive your Aura agency control plane from Claude — approvals, snapshots, connections, runs — over the Aura MCP gateway. Teaches the `aura__*` control-plane tools (list_pending_approvals, get_action, list_snapshots, list_connections, list_runs, client_summary, reject_action, restore_snapshot, rollback_run, approve_action) and the connect flow: mint a `canManage` management token in Aura, point an MCP HTTP client at the fleet gateway, and read/govern the agency itself — not just the managed sites. This is a thin connector, not a standalone server: all auth, policy, approval-gating, and audit live in the Aura gateway (single enforcement point). Requires an Aura account (app.my-aura.app) — no standalone value without one. Use when the user references Aura, aura-mcp, the Aura control plane, `/aura-mcp`, or runs `aura__*` tools; covers minting a management token, wiring the MCP client config, and the governance/safety model (human-tap approvals by default, self-approval guard, client-wide-only reverts). SKIP for the outward site/builder/content/infra tools (those are the fleet gateway's other tool groups), and for non-Aura MCP work.
permissions:
  network:
    - "The Aura fleet MCP gateway over HTTPS (default https://app.my-aura.app/api/mcp/fleet) — JSON-RPC tools/list + tools/call, authenticated by a Bearer aura_ management token. No other host is contacted."
  filesystem:
    - "Writes an MCP client config (.mcp.json in the current working directory, or the client's config file) that embeds the aura_ management token; appends .mcp.json to .gitignore there."
  env:
    - "AURA_MCP_URL (optional — override the gateway URL; defaults to https://app.my-aura.app/api/mcp/fleet)"
    - "AURA_MANAGEMENT_TOKEN (optional — supply the aura_ management token instead of pasting it into the config)"
---

# Aura MCP — control-plane skill

You are operating **Aura itself** over MCP — the agency control plane (approvals,
snapshots, connections, runs), not the managed WordPress sites. The `aura__*` tools
are served by Aura's **fleet gateway** (`/api/mcp/fleet`) and are visible **only** to a
**management token** (`canManage`). Everything is scoped to that token's
organization + client.

> **This is a thin connector, not a server.** There is nothing to install and no logic
> to run locally. Auth, tool policy, approval-gating, and the audit trail all live in the
> Aura gateway — the single enforcement point. The skill just teaches the tools and wires
> the client. Never re-implement a gateway; that would fork the governance.

## 🛑 First action — ASK, then connect

When this skill is invoked, do **not** start calling tools. First establish the connection,
then ask what the user wants.

1. **Is a management token wired?** If an MCP client is already connected to the Aura
   gateway and `tools/list` shows `aura__*` tools, you're ready — skip to the menu.
2. **If not,** walk the user through **[Connect](references/connect.md)** — mint a
   `canManage` token in Aura and add the MCP config. Do this only on explicit confirmation;
   the token is a credential.

Then present the menu and wait:

```
What would you like to do with your Aura agency?

  READ (safe, always available on a management token)
  1. Show pending approvals            → aura__list_pending_approvals
  2. Inspect one action                → aura__get_action
  3. List snapshots (rollback points)  → aura__list_snapshots
  4. List provider connections         → aura__list_connections
  5. List recent agent runs            → aura__list_runs
  6. Client situational summary        → aura__client_summary

  WRITE (governed — see safety notes)
  7. Reject a pending action           → aura__reject_action        (safe: only denies)
  8. Restore a page snapshot           → aura__restore_snapshot     (revert; client-wide token)
  9. Roll back a whole run             → aura__rollback_run         (revert; client-wide token)
 10. Approve + run a pending action    → aura__approve_action       (most privileged; opt-in)
```

## Tool reference

Full schemas, arguments, return shapes, and worked examples: **[references/tools.md](references/tools.md)**.

Quick map (R = read, W = write, W! = high-risk revert/execute):

| Tool | Kind | What it does |
|---|---|---|
| `aura__list_pending_approvals` | R | Agent actions awaiting human approval in scope |
| `aura__get_action` | R | One action's full status / params / result |
| `aura__list_snapshots` | R | Page snapshots (rollback points), filterable by resource/post |
| `aura__list_connections` | R | Provider connections + validation status (never credentials) |
| `aura__list_runs` | R | Recent runs (actions grouped by `runId`) |
| `aura__client_summary` | R | One-shot counts: resources, connections, pending, snapshots |
| `aura__reject_action` | W | Deny a pending action so it never runs (safe — only denies) |
| `aura__restore_snapshot` | W! | Roll a page back to a snapshot (client-wide token only) |
| `aura__rollback_run` | W! | Unwind a whole run, newest snapshot first (client-wide token only) |
| `aura__approve_action` | W! | Approve **and run** a pending action (opt-in + self-approval guard) |

## Governance & safety (read before any write)

Full model: **[references/safety.md](references/safety.md)**. The load-bearing rules:

- **Approvals are human-tap by default.** `aura__approve_action` **executes** the queued
  write. It ships behind THREE gates: the token must (a) be `canManage`, (b) explicitly
  allow-list `aura__approve_action` in its `allowedTools`, and (c) the organization must
  have opted into machine-approve (off by default). If any is missing, approving stays in
  the Aura UI — surface + comment on the action instead.
- **No self-approval.** A token may not approve an action it (its creator) requested. The
  gateway refuses it server-side. Don't try to work around it.
- **Reverts need a client-wide token.** `aura__restore_snapshot` and `aura__rollback_run`
  refuse a pack-scoped token (a run/snapshot chain can span resources outside the pack).
- **Explicit opt-in for the dangerous three.** `restore_snapshot`, `rollback_run`, and
  `approve_action` are NOT granted by the "empty `allowedTools` = unrestricted" default —
  the token must name them. Reads and `reject_action` (which can only deny) ride the default.
- **Every `aura__*` call is audited** (`AgentUsageEvent`); machine-approvals are flagged
  distinctly as approved-via-MCP. Assume the trail sees everything you do.

## Positioning (say this if asked what aura-mcp is)

The **capstone** of the Aura Design Engine family — the product that ties the others
together: drive your whole agency (governance, approvals, rollbacks) from your editor over
MCP. Unlike the site/host skills, it has **no standalone value without an Aura account** —
it governs the fleet the other tools operate on.
