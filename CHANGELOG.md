# Changelog

All notable changes to the **aura-mcp** skill are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this kit tracks
the `aura__*` control-plane tool surface shipped by the Aura fleet gateway.

## [0.1.0] — 2026-07-09

Initial public release — the P5 (distribution) step of the aura-mcp plan: a skill + thin
connector for the control-plane tools already shipped in the Aura gateway (planks P1–P4).

### Added
- **SKILL.md** — the cheat sheet Claude reads: first-action protocol, tool map, menu, and the
  governance rules that gate the writes.
- **references/tools.md** — all 10 `aura__*` tools with arguments, return shapes, and error
  codes:
  - Reads: `list_pending_approvals`, `get_action`, `list_snapshots`, `list_connections`,
    `list_runs`, `client_summary`.
  - Writes: `reject_action` (safe), `restore_snapshot`, `rollback_run`, `approve_action`.
- **references/connect.md** — mint a `canManage` management token (Fleet → Agent Tokens →
  "allow manage") and wire an MCP client (Claude Code / Desktop / Cursor) at
  `/api/mcp/fleet` with a `Bearer aura_` token.
- **references/safety.md** — the governance model: human-tap-by-default approvals, the
  layered `approve_action` gates (canManage + allow-list + org opt-in + known-owner +
  self-approval guard), client-wide-only reverts, and scope enforcement.
- **docs/QUICKSTART.md** — 5-minute connect.

### Notes
- This is a **connector**, not a server — all auth, policy, approval-gating, and audit live
  in the Aura gateway (single enforcement point). No governance is re-implemented here.
- Requires an Aura account; no standalone value without one.
