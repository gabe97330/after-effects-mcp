# After Effects MCP — Claude Code Context

## Project Purpose

A Model Context Protocol (MCP) server that lets AI assistants control Adobe After Effects.
Communication works via a file-based handshake: the Node.js server writes commands to
`~/Documents/ae-mcp-bridge/ae_command.json`, and an ExtendScript panel running inside After
Effects polls for them, executes them, and writes results to `ae_mcp_result.json`.

## Current Task

**Validate macOS support.** The macOS compatibility PR has already been merged into this fork
(commit `5b89524`, "feat: add macOS compatibility and AE 2026 support"). Your job is to
verify the implementation is correct and run as much of the validation as possible.

See `docs/macos-validation.md` for the full step-by-step plan, including which steps can be
automated and which require a running copy of After Effects.

## Key macOS Changes (already applied)

| File | What changed |
|------|-------------|
| `install-bridge.js` | Detects `process.platform === 'darwin'`; targets `/Applications/Adobe After Effects [VERSION]/Scripts/ScriptUI Panels/` (no `Support Files/` prefix that Windows needs); falls back to `sudo cp` instead of PowerShell elevation |
| `src/index.ts` | `getAETempDir()` uses `os.homedir() + /Documents/ae-mcp-bridge` — avoids `/tmp` which macOS sandboxes away from the app process |
| `src/scripts/mcp-bridge-auto.jsx` | Uses `Folder.myDocuments` (ExtendScript cross-platform API) for the bridge dir; creates a floating palette for AE 2025+ (dockable panels not supported there) |

## Build & Run

```bash
npm install        # also runs `npm run build` via postinstall
npm run build      # tsc + copies jsx scripts to build/scripts/
npm run install-bridge   # copies panel to AE ScriptUI Panels folder
npm start          # starts the MCP server on stdio
```

Build artifacts:
- `build/index.js` — MCP server entry point
- `build/scripts/mcp-bridge-auto.jsx` — After Effects panel script

## Architecture

```
AI Client (Claude/Cursor)
    │  MCP protocol (stdio)
    ▼
Node.js MCP Server (build/index.js)
    │  writes ae_command.json
    ▼
~/Documents/ae-mcp-bridge/
    │  ae_command.json   ← commands in
    │  ae_mcp_result.json ← results out
    ▼
After Effects (mcp-bridge-auto.jsx panel)
    polls every 2s, executes via ExtendScript
```

## MCP Config (Claude Desktop / Cursor)

```json
{
  "mcpServers": {
    "AfterEffectsMCP": {
      "command": "node",
      "args": ["/absolute/path/to/after-effects-mcp/build/index.js"]
    }
  }
}
```

## Repo Layout

```
src/
  index.ts                     # MCP server
  scripts/
    mcp-bridge-auto.jsx        # AE panel (main bridge, 1300 lines)
    applyEffect.jsx
    createComposition.jsx
    createTextLayer.jsx
    createShapeLayer.jsx
    createSolidLayer.jsx
    getLayerInfo.jsx
    getProjectInfo.jsx
    listCompositions.jsx
    setLayerProperties.jsx
install-bridge.js               # installs panel into AE
docs/
  macos-validation.md           # validation & smoke test plan
```

## Notes

- `npm run build` is run automatically on `npm install` via `postinstall`.
- The server communicates only via stdio (no HTTP port). Do not attempt to curl it.
- Commands time out after 5 seconds if After Effects does not respond.
- After Effects must have "Allow Scripts to Write Files and Access Network" enabled
  (After Effects > Settings > Scripting & Expressions on macOS).
