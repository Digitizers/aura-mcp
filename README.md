# Aura MCP — Aura control-plane Claude skill

![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-d97757)
![OpenClaw Skill](https://img.shields.io/badge/OpenClaw-Skill-purple)
![Aura](https://img.shields.io/badge/Aura-control_plane-6d28d9)
![License: MIT](https://img.shields.io/badge/License-MIT-green)
![Version](https://img.shields.io/badge/version-0.1.1-blue)

Drive your **Aura agency control plane from Claude** — approvals, snapshots, connections,
runs — over MCP. Not the managed sites (the fleet's other tools already do those): **Aura
itself**. See what's queued for approval, inspect actions and runs, list rollback points and
provider connections, reject or (opt-in) approve pending writes, and roll back a page or a
whole run — from Claude Code, Claude Desktop, or Cursor.

> **The short version:** mint a management token in Aura, point an MCP client at the Aura
> fleet gateway, and Claude can now govern your agency — with every action scoped, gated, and
> audited exactly as it is in the Aura UI. This is a **thin connector**: nothing to install,
> no server to run. All the governance lives in Aura's gateway (the single enforcement point).

> **Requires an Aura account** ([app.my-aura.app](https://app.my-aura.app)). Unlike the other
> family skills, aura-mcp has **no standalone value** — it governs the fleet the others operate
> on.

---

## Part of the Aura Design Engine

These are the skills behind [**Aura**](https://my-aura.app) — one AI web-agency lifecycle you
can run standalone or orchestrate across a whole client fleet from a single dashboard.

| Stage | Skill | Role |
| --- | --- | --- |
| 🎨 Build | [siteagent-elementor-studio](https://github.com/Digitizers/siteagent-elementor-studio) | Design & build sites inside Elementor |
| 🔎 Audit + Content | [wordpress-api-pro](https://github.com/Digitizers/wordpress-api-pro) | REST content ops, SEO & site audits |
| 🖥 Host | [cloudways-mcp](https://github.com/Digitizers/cloudways-mcp) · [hostinger-mcp](https://github.com/Digitizers/hostinger-mcp) | Provision & operate the infrastructure |
| 🛡 Govern | [**aura-mcp** ← you are here](https://github.com/Digitizers/aura-mcp) | Drive the agency control plane — approvals, snapshots, rollbacks — over MCP |

The others acquire and operate; **aura-mcp is the capstone** — it governs the whole fleet from
your editor.

## What's inside

```
aura-mcp/
├── files/
│   ├── SKILL.md                 ← The cheat sheet Claude reads
│   └── references/
│       ├── tools.md             ← All 10 aura__* tools: schemas, args, returns
│       ├── connect.md           ← Mint a management token + wire the MCP client
│       └── safety.md            ← Governance model: approvals, self-approval guard, reverts
├── docs/
│   └── QUICKSTART.md            ← 5-minute connect
├── LICENSE                      ← MIT (the skill kit; Aura itself is a separate product)
└── CHANGELOG.md
```

## Quickstart

1. **Mint a management token** — Aura → **Fleet** → Agent Tokens → check **"allow manage"**
   (`canManage`). Copy the `aura_…` token (shown once).
2. **Wire an MCP client** — point it at the gateway with the token:

   ```json
   {
     "mcpServers": {
       "aura": {
         "type": "http",
         "url": "https://app.my-aura.app/api/mcp/fleet",
         "headers": { "Authorization": "Bearer aura_YOUR_MANAGEMENT_TOKEN" }
       }
     }
   }
   ```

3. **Verify** — ask Claude to run `aura__client_summary`. Counts back = connected.

Full steps (Claude Code / Desktop / Cursor, scoping, high-risk-write opt-in):
[`files/references/connect.md`](files/references/connect.md).

## The tools

| Tool | Kind | What it does |
|---|---|---|
| `aura__list_pending_approvals` | read | Agent actions awaiting human approval |
| `aura__get_action` | read | One action's full status / params / result |
| `aura__list_snapshots` | read | Page snapshots (rollback points) |
| `aura__list_connections` | read | Provider connections + validation status (never credentials) |
| `aura__list_runs` | read | Recent runs (actions grouped by `runId`) |
| `aura__client_summary` | read | One-shot counts: resources, connections, pending, snapshots |
| `aura__reject_action` | write | Deny a pending action (safe — only denies) |
| `aura__restore_snapshot` | write! | Roll a page back to a snapshot (client-wide token) |
| `aura__rollback_run` | write! | Unwind a whole run (client-wide token) |
| `aura__approve_action` | write! | Approve **and run** a pending action (opt-in + guards) |

Schemas + examples: [`files/references/tools.md`](files/references/tools.md).

## Safety in one paragraph

Approvals stay **human-tap by default**. `aura__approve_action` executes a queued write and
ships behind three gates — `canManage`, explicit allow-listing, and an org opt-in — plus a
server-side **self-approval guard** (a token can't approve an action it requested). Reverts
(`restore_snapshot` / `rollback_run`) require a **client-wide** token. Every `aura__*` call is
audited. Full model: [`files/references/safety.md`](files/references/safety.md).

## Why a connector, not a server

Aura's gateway (`/api/mcp/fleet`) is the **single enforcement point** for auth, tool policy,
approval-gating, and audit. A standalone re-implementation would fork that governance — the one
thing that must never drift. aura-mcp is a dumb pointer at the gateway URL + token; all the
logic stays in Aura.

## License

MIT — for this skill kit (the docs and config that teach Claude the tools). Aura itself is a
separate hosted product. The `aura__*` tools are served by your Aura account's gateway.

### Windows note

The plugin ships its skill through a git **symlink** (`skills/` → the in-repo
source). On Windows, enable Developer Mode and set
`git config --global core.symlinks true` **before** cloning or installing —
the plugin cache clone inherits it. Changing the config does not repair an
existing checkout (the repo may have recorded `core.symlinks=false` locally).
To repair one, run these two commands inside it (the second re-materializes
only the plugin's symlink entries, so nothing else in your working tree is
touched):

    git config core.symlinks true
    git checkout -- skills/ .claude/skills/

Or simply re-clone. WSL also works. macOS/Linux need nothing.
