# macOS Validation & Smoke Test Plan

This document is the authoritative checklist for validating macOS support in this repo.
Steps are ordered from fully-automatable to requiring a running copy of After Effects.

---

## Phase 1 — Build Verification (no After Effects needed)

These steps can be run entirely from the terminal.

### 1.1 Install dependencies and build

```bash
npm install
```

**Expected:** No errors. The `postinstall` hook runs `npm run build` automatically.

**Verify build artifacts exist:**

```bash
ls build/index.js
ls build/scripts/mcp-bridge-auto.jsx
```

Both files must be present. If either is missing, run `npm run build` and check for
TypeScript errors.

### 1.2 Confirm platform detection logic

Run a quick Node.js one-liner to confirm `install-bridge.js` will use the macOS branch:

```bash
node -e "console.log('platform:', process.platform); console.log('isMac:', process.platform === 'darwin')"
```

**Expected on macOS:**
```
platform: darwin
isMac: true
```

### 1.3 Verify bridge directory can be created

```bash
node -e "
const os = require('os');
const fs = require('fs');
const path = require('path');
const bridgeDir = path.join(os.homedir(), 'Documents', 'ae-mcp-bridge');
fs.mkdirSync(bridgeDir, { recursive: true });
console.log('Bridge dir OK:', bridgeDir);
console.log('Writable:', fs.accessSync(bridgeDir, fs.constants.W_OK) === undefined ? 'yes' : 'yes');
"
```

**Expected:** Prints the path to `~/Documents/ae-mcp-bridge` with no error.
This is the critical directory — both the Node.js server and the AE panel exchange files here.

### 1.4 Confirm target AE installation path is detected

```bash
node -e "
const fs = require('fs');
const paths = [
  '/Applications/Adobe After Effects 2026',
  '/Applications/Adobe After Effects 2025',
  '/Applications/Adobe After Effects 2024',
  '/Applications/Adobe After Effects 2023',
  '/Applications/Adobe After Effects 2022',
  '/Applications/Adobe After Effects 2021',
];
const found = paths.find(p => fs.existsSync(p));
if (found) {
  console.log('Found AE at:', found);
  console.log('Panel target:', found + '/Scripts/ScriptUI Panels/');
} else {
  console.log('No AE installation found in standard locations.');
  console.log('Manual install will be required.');
}
"
```

**Expected:** Prints the path to the installed version of After Effects.
If nothing is found, the `install-bridge.js` script will exit with an error — manual
panel installation will be needed (see Phase 2 fallback).

### 1.5 Confirm macOS path structure (not Windows)

```bash
node -e "
const path = require('path');
// macOS: no 'Support Files' prefix
const mac  = '/Applications/Adobe After Effects 2025/Scripts/ScriptUI Panels';
// Windows (for reference): 'C:\\...\\Support Files\\Scripts\\ScriptUI Panels'
console.log('macOS panel path:', mac);
console.log('Correct — no Support Files prefix on macOS');
"
```

Cross-check that `install-bridge.js` line 58-60 uses the macOS path:

```bash
node -e "
const src = require('fs').readFileSync('install-bridge.js', 'utf8');
const hasCorrectMacPath = src.includes(\"path.join(afterEffectsPath, 'Scripts', 'ScriptUI Panels')\");
const hasWinPath = src.includes(\"'Support Files', 'Scripts', 'ScriptUI Panels'\");
console.log('macOS path (no Support Files):', hasCorrectMacPath ? 'PRESENT' : 'MISSING');
console.log('Windows path (Support Files):', hasWinPath ? 'PRESENT' : 'MISSING');
"
```

**Expected:** Both lines print the appropriate path. macOS path must NOT include `Support Files`.

### 1.6 Smoke-test the file handshake (no AE)

This simulates one round of the handshake without After Effects.

```bash
node -e "
const os = require('os');
const fs = require('fs');
const path = require('path');
const bridgeDir = path.join(os.homedir(), 'Documents', 'ae-mcp-bridge');
fs.mkdirSync(bridgeDir, { recursive: true });

// Write a command file (as the MCP server would)
const cmd = {
  command: 'getProjectInfo',
  args: {},
  timestamp: new Date().toISOString(),
  status: 'pending'
};
const cmdPath = path.join(bridgeDir, 'ae_command.json');
fs.writeFileSync(cmdPath, JSON.stringify(cmd, null, 2));
console.log('Wrote command to:', cmdPath);
console.log(fs.readFileSync(cmdPath, 'utf8'));

// Simulate AE panel writing a result
const result = {
  _commandExecuted: 'getProjectInfo',
  status: 'completed',
  data: { name: 'Test Project', fakeSimulation: true },
  timestamp: new Date().toISOString()
};
const resPath = path.join(bridgeDir, 'ae_mcp_result.json');
fs.writeFileSync(resPath, JSON.stringify(result, null, 2));
console.log('Simulated result at:', resPath);
console.log(fs.readFileSync(resPath, 'utf8'));
"
```

**Expected:** Both files are written and read back without error.

---

## Phase 2 — Panel Installation (After Effects required)

### 2.1 Run the installer

```bash
npm run install-bridge
```

**Expected output:**
```
Installing bridge script to /Applications/Adobe After Effects [VERSION]/Scripts/ScriptUI Panels/mcp-bridge-auto.jsx...
Bridge script installed successfully!

Important next steps:
1. Open After Effects
2. Go to After Effects > Settings > Scripting & Expressions
3. Enable "Allow Scripts to Write Files and Access Network"
4. Restart After Effects
5. Open the bridge panel: Window > mcp-bridge-auto.jsx
```

If the direct copy fails due to permissions, the script will automatically retry with `sudo cp`.

**Verify installation:**
```bash
# Replace 2025 with your installed version
ls -la "/Applications/Adobe After Effects 2025/Scripts/ScriptUI Panels/mcp-bridge-auto.jsx"
```

### 2.2 Manual install fallback

If `npm run install-bridge` cannot find your AE installation (non-standard path), copy manually:

```bash
# Build first if not already done
npm run build

# Copy manually — update the version year as needed
sudo cp build/scripts/mcp-bridge-auto.jsx \
  "/Applications/Adobe After Effects 2025/Scripts/ScriptUI Panels/mcp-bridge-auto.jsx"
```

---

## Phase 3 — After Effects Panel (After Effects must be running)

### 3.1 Enable script permissions

1. Open After Effects
2. **After Effects > Settings > Scripting & Expressions** (macOS — not Edit > Preferences)
3. Check **"Allow Scripts to Write Files and Access Network"**
4. Restart After Effects

### 3.2 Open the bridge panel

1. In After Effects: **Window > mcp-bridge-auto.jsx**
2. The panel should appear as a **floating palette** (AE 2025+) or dockable panel (older)
3. Status should show: `Waiting for commands...`
4. The "Auto-run commands" checkbox should be enabled

**If the panel does not appear in the Window menu:** The file was not copied to the correct
location. Re-run Phase 2.

### 3.3 Verify bridge directory is created by the panel

With the panel open in AE, check from terminal:

```bash
ls ~/Documents/ae-mcp-bridge/
```

**Expected:** Directory exists (the panel creates it on startup via `Folder.myDocuments`).

---

## Phase 4 — End-to-End File Handshake

### 4.1 Manual command injection

With the AE panel running, write a command from terminal and observe the panel process it:

```bash
cat > ~/Documents/ae-mcp-bridge/ae_command.json << 'EOF'
{
  "command": "getProjectInfo",
  "args": {},
  "timestamp": "2026-01-01T00:00:00.000Z",
  "status": "pending"
}
EOF
```

Within ~3 seconds (panel polls every 2s), check for a result:

```bash
sleep 3 && cat ~/Documents/ae-mcp-bridge/ae_mcp_result.json
```

**Expected:** JSON object containing project information (name, frame rate, items, etc.).

### 4.2 Test list-compositions

```bash
cat > ~/Documents/ae-mcp-bridge/ae_command.json << 'EOF'
{
  "command": "listCompositions",
  "args": {},
  "timestamp": "2026-01-01T00:00:00.001Z",
  "status": "pending"
}
EOF
sleep 3 && cat ~/Documents/ae-mcp-bridge/ae_mcp_result.json
```

**Expected:** JSON array of compositions (may be empty if no project is open — that is fine,
the important thing is a valid JSON response, not an error).

---

## Phase 5 — Full MCP Integration

### 5.1 Configure MCP client

Add to your Claude Desktop or Cursor MCP config:

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

Restart the MCP client after saving.

### 5.2 Verify tools are registered

Ask the AI assistant:
> "What After Effects tools do you have available?"

**Expected:** The assistant lists tools including `create-composition`, `run-script`,
`get-results`, `setLayerKeyframe`, `setLayerExpression`, `apply-effect`, etc.

### 5.3 End-to-end tool call

With After Effects open and the panel running, ask:
> "Get the current After Effects project info"

**Expected:** The assistant calls the MCP tool, the panel processes the command, and returns
actual project data (or a clear "no project open" message — not a timeout or file error).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `install-bridge.js` can't find AE | Non-standard install path | Manual copy (Phase 2.2) |
| Panel not in Window menu | Wrong destination path | Verify path has no `Support Files/` prefix |
| `ae_mcp_result.json` never appears | Scripting not enabled | Phase 3.1 — enable in AE Settings |
| Result file is stale (>30s old) | Panel not polling | Re-open panel, check "Auto-run commands" checkbox |
| `~/Documents/ae-mcp-bridge/` missing | Panel not opened yet | Open panel in AE first (Window menu) |
| MCP tools time out | AE not running or panel closed | Ensure both are running before invoking tools |
| `sudo cp` password prompt | Panel dir permissions | Enter macOS password when prompted |

---

## Summary Checklist

- [ ] `npm install` completes without errors
- [ ] `build/index.js` exists
- [ ] `build/scripts/mcp-bridge-auto.jsx` exists
- [ ] `process.platform === 'darwin'` on this machine
- [ ] `~/Documents/ae-mcp-bridge/` is writable
- [ ] AE installation found at `/Applications/Adobe After Effects [VERSION]/`
- [ ] `npm run install-bridge` copies panel to correct macOS path (no `Support Files/`)
- [ ] AE scripting permission enabled
- [ ] Panel opens as floating palette in AE 2025+
- [ ] Manual command injection produces a result within 3 seconds
- [ ] Full MCP tool call (`get-results` or `create-composition`) succeeds end-to-end
