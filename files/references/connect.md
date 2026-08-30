# Connect Claude to the Aura control plane

Two steps: **mint a management token** in Aura, then **point an MCP client** at the gateway.
The token is a credential — treat it like a password. Do the wiring only on the user's
explicit confirmation.

## 1. Mint a `canManage` management token

1. Sign in to Aura at **https://app.my-aura.app**.
2. Open the **Fleet** page → **Agent Tokens**.
3. Create a new token and check **"allow manage"** — this is the `canManage` flag that
   unlocks the `aura__*` control-plane tools. Grant it sparingly; it is strictly higher
   privilege than a normal execution token.
4. **Scope it.** The token is bound to your organization + a client. To restrict it to one
   Site Group, set its pack — but note the **reverts** (`aura__restore_snapshot`,
   `aura__rollback_run`) require a **client-wide** token and will refuse a pack-scoped one.
5. **For the high-risk writes**, add their names to the token's **allowed tools** — they are
   NOT granted by default:
   - `aura__restore_snapshot`
   - `aura__rollback_run`
   - `aura__approve_action` (also needs org opt-in — see [safety.md](safety.md))

   **A read-only token must use an explicit allow-list — do NOT leave it empty.** An empty
   `allowedTools` means *unrestricted*: a `canManage` token can then call every read **and
   mutating** fleet tool it can reach (site / infra / content ops), excepting only the three
   high-risk `aura__*` writes listed above. For a genuinely read-only (+ deny) token, allow-list
   only the read tools you need — `aura__list_pending_approvals`, `aura__get_action`,
   `aura__list_snapshots`, `aura__list_connections`, `aura__list_runs`, `aura__client_summary` —
   plus `aura__reject_action` (deny-only, can never execute). Leave `allowedTools` empty **only**
   for a fully-trusted operator token that is meant to run everything.
6. **Copy the token** — it's shown **once**, in the form `aura_` followed by 48 hex
   characters. Store it in a secret manager, not in a committed file.

## 2. Wire the MCP client

The gateway speaks MCP over HTTP at **`https://app.my-aura.app/api/mcp/fleet`**, authenticated
with `Authorization: Bearer aura_…`. On a management token, `tools/list` includes the
`aura__*` tools alongside the fleet's site/infra/content tools.

### Claude Code — env var (zero-config, devices and cloud sessions)

This repo commits a placeholder-only `.mcp.json` whose `Authorization` header reads
`Bearer ${AURA_MCP_TOKEN:-}`. Set that variable — in your shell profile on a device, or in the
claude.ai cloud environment's environment variables for web/phone sessions — and the `aura`
connection authenticates automatically. While the variable is unset the config still parses
(the `:-` default) and the header goes out with an empty bearer, so the gateway answers
**401** and the client reports `AUTH_HEADER_REJECTED` — it does *not* quietly show as
unavailable. That is expected until you provide the token, and since Aura #454 the 401 body
says which side is at fault ("no credential presented" vs "agent token rejected"), so an
unset variable no longer reads as a bad token. Never put a real token in the tracked
`.mcp.json`. Cloud environments with a restricted network policy must allow
`app.my-aura.app`.

To see what your client actually resolved the header to — the fastest way to tell a
substitution problem from a credential one — run `claude mcp get plugin:aura-mcp:aura`
(the value is redacted in the output; what you are checking is `Status`).

In a project of your own, the equivalent explicit config is:

```json
{
  "mcpServers": {
    "aura": {
      "type": "http",
      "url": "https://app.my-aura.app/api/mcp/fleet",
      "headers": {
        "Authorization": "Bearer aura_YOUR_MANAGEMENT_TOKEN"
      }
    }
  }
}
```
If you inline a real token like this, keep that `.mcp.json` out of version control
(gitignore it) — or better, keep the `${AURA_MCP_TOKEN:-}` placeholder and export the
token in your environment.

### Claude Desktop — `claude_desktop_config.json`
(`%APPDATA%\Claude\` on Windows, `~/Library/Application Support/Claude/` on macOS)
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

### Cursor — `.cursor/mcp.json`

Cursor interpolates environment variables in MCP config with `${env:VAR}` — use it so the
token stays out of the file:

```json
{
  "mcpServers": {
    "aura": {
      "url": "https://app.my-aura.app/api/mcp/fleet",
      "headers": { "Authorization": "Bearer ${env:AURA_MCP_TOKEN}" }
    }
  }
}
```

### Self-hosted Aura
Replace the host with your deployment's app URL, keeping the `/api/mcp/fleet` path. Override
via `AURA_MCP_URL` if the skill's tooling reads it.

## 3. Verify

Ask the client to list tools, or call `aura__client_summary`. If you see the `aura__*` tools
and a summary comes back, you're connected. If `aura__*` tools are **absent**, the token
almost certainly lacks `canManage` — re-mint with "allow manage" checked.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| No `aura__*` tools in `tools/list` | Token isn't `canManage`. Re-mint with "allow manage". |
| `Connected · tools fetch failed` / `Request timed out`, or the client sits at `connecting…` | **Not an auth problem — auth already passed.** `tools/list` asks every connected site for its tools, and a large or partly unreachable fleet used to outrun the request (Aura #454). Fixed gateway-side; if you still see it, the fleet has sites that answer very slowly. |
| `Unauthorized: no credential presented` | The header arrived with an empty bearer — `AURA_MCP_TOKEN` is unset (or empty) in the environment that launched the client. Nothing wrong with your token. |
| `Unauthorized: malformed credential` | The header is not `Bearer aura_<48 hex>` — an unexpanded `${AURA_MCP_TOKEN}` literal, a truncated paste, or a missing space after `Bearer`. A client-config problem, not a token one. |
| `Unauthorized: agent token rejected` | *Now* it's the token: unknown, revoked, or expired. Re-mint in Aura → Fleet → Agent Tokens. |
| `Unauthorized: invalid or expired agent token` | The pre-#454 message, which meant *any* of the three above. If you see it, the gateway predates the split — use the literal-token control below to tell them apart. |
| Write returns `PACK_SCOPED_TOKEN` | Reverts need a client-wide token; this one is pack-scoped. |
| Write returns `ORG_OPT_IN_REQUIRED` | Machine-approve is off for the org — approve in the Aura UI, or an admin enables it. |
| `aura__approve_action` not callable | Add it to the token's allowed-tools (explicit opt-in), and confirm org opt-in. |

### Telling a substitution problem from a bad token

If a 401 leaves you unsure whether the token is wrong or never reached the header, don't
re-mint first — re-minting is the expensive guess, and it was the wrong one in
[aura-mcp#5](https://github.com/Digitizers/aura-mcp/issues/5).

Two controls, in order of cost:

1. **Ask the client what it resolved.** `claude mcp get plugin:aura-mcp:aura` prints the
   server's status and headers (the credential is redacted). A `Status` of `Connected` means
   the header was substituted and accepted — whatever else is failing is not auth.
2. **Replay the token by hand.** This is the ground truth for the credential itself:

   ```bash
   curl -sS -X POST https://app.my-aura.app/api/mcp/fleet \
     -H "Authorization: Bearer $AURA_MCP_TOKEN" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json, text/event-stream" \
     -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"1"}}}'
   ```

   A 200 with `serverInfo` proves the token is good and the problem is on the client side of
   the header. A 401 names which of the three failures it is.

   Send `initialize` first. `tools/list` is answered on its own, but it fans out across the
   whole fleet, so on a large fleet it is the slowest call to probe with — not a good first
   test of a credential.

A token pasted literally into a config as a comparison test is a legitimate control, but a
`canManage` token in plaintext is not a resting state — remove it once it has told you what
you needed.
