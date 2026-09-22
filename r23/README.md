# r23 evidence — live CodeArts guard-string + MCP-sanction captures (2026-09-22)

## Capture method (upgraded from r22)
Every `x11/*.png` is a TRUE X11 pixel screenshot (`import -window root` on the Xvfb :99 display)
of an `xterm` attached to the live tmux session running the REAL `codearts` CLI v26.9.4
(model huaweicloud-maas/GLM-5.1, auth via CODEARTS_CLI_AK/SK env). No ansi2html, no re-render:
the PNG is what the X server rasterized for the live terminal. The matching `tui-frames/*.txt`
carries the raw `tmux capture-pane -e -p` bytes of (approximately) the same instant, so every
pixel frame can be checked against terminal bytes. The green bottom bar is tmux's own status
line — the terminal is a tmux client, honestly shown.

## Guard path
Real front door only: plugin `~/.codeartsdoer/plugin/onyx-guard.ts` → bundled onyx-scanner
`hooks --source=codearts` → `http://localhost:8080` (Kong) → sanitization service. Policy 27
("CodeArts unsanctioned shell block (e2e r22)") extended this round via the real crud front door
(`PUT /api/crud/v1/policies/`, SSO browser session) with two deterministic Block rules:
- `custom_rule` "Block CodeArts prompt probe" — exact keyword `onyx-guard-prompt-probe-r23` (input).
- `tool_execution` "Block CodeArts tool-output probe" — regex `tool.outputs.text = onyx-guard-output-probe-r23`
  (fires only on the post-tool lane: the pre-tool payload carries no outputs).
Every DENY cites a fresh per-hit ONYX_VIOLATION_ID minted by the live policy engine.

## MCP-sanction captures (21/22/23)
The MCP host is the OFFICIAL @modelcontextprotocol/server-everything reference server
v2026.8.31 served over streamable HTTP on this host (127.0.0.1:3001/mcp) — a genuine MCP
protocol server (real initialize handshake, real tools), honestly run locally; the SANCTION
VERDICTS are the real access-control front door's
(`/access-control/authorize/.../mcp/...`; rule set via `PUT /api/crud/v1/access-control/bulk`).
Registered in `~/.codeartsdoer/codearts_cli.json` as remote server `lab-everything`.

**CodeArts finding (default mode):** CodeArts 26.9.4 ships an MCP "lazy loader"
(`mcp-tool-lazy-loader-plugin`) that is ON by default and forces every MCP call through
meta-tools (`tool_search`/`tool_describe`/`tool_call`; the binary rejects direct calls:
`MCP tool "X" is not directly callable`). The hook then receives `tool_name="tool_call"`,
the opencode identity resolver cannot match a server prefix, and the sanction lane FAILS OPEN
with `unresolved_prefix` telemetry (`authorize_request_sent:false`) — see
`mcp/20-default-mode-failopen-telemetry.txt`. With the agent's own supported toggle
`ENABLE_TOOL_LAZY_LOADING=false` ("passing all tools through" — gate read from the shipped
binary), direct MCP tool names flow and the full sanction lane works: resolve → authorize →
allow/deny (`mcp/25-sanction-telemetry.txt`). The captures 21/22/23 were made in that mode.
The default-mode fail-open is a REAL coverage gap, reported honestly in the PR ledger
(owner decision: close via meta-tool unwrapping in the resolver, or accept as the documented
unresolved-prefix fail-open class).

## Files
- x11/10-promptblock-toast.png + tui-frames/10-*.txt — formatBlockReplacementPrompt live + toast (violation 37465270…)
- x11/11-promptblock-settled.png + tui-frames/11-*.txt — + model relaying the block, Build 9.3s
- x11/12-outputblock-toast-settled.png + tui-frames/12-*.txt — blockedToolOutputNotice flow live + toast (violation 7a23074a…)
- 08-session-store-redacted-output.json — the agent's OWN session store row: state.output = the verbatim
  blockedToolOutputNotice the model received; metadata.output = the local sandbox text. The asymmetry IS the redaction.
- 07-keylog-toasts.txt — codearts kernel log bus-publish lines for both showBlockToast events
- 06-*.request/response.json — front-door curl verification of both new lanes (block + allow controls)
- x11/21-mcp-unsanctioned-deny-toast.png + tui-frames/21-*.txt — UNSANCTIONED deny live (+ toast): the MCP-sanction
  deny string with the real backend reason; same frame shows the earlier default-state allow
- x11/22-mcp-sanctioned-allow.png + tui-frames/22-*.txt — SANCTIONED allow live (Raw result: Echo: sanctioned now)
- x11/23-mcp-default-allow.png + tui-frames/23-*.txt — first call, Discovered/default → allow (ingest trigger)
- tui-frames/13-*, 24-* — full-session raw scrollbacks
- mcp/20-*, mcp/25-* — scanner mcp_identity_outcome telemetry (fail-open default mode; resolved allow/deny pass-through mode)
- 01-policy27-updated.png — the policy PUT via the real UI session
