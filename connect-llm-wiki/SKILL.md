---
name: connect-llm-wiki
description: >-
  Wire the LLM Wiki desktop app's local knowledge-base API into an AI coding agent as an MCP tool
  source — covers both `pi` (needs the `pi-mcp-adapter` extension, config at
  `~/.config/mcp/mcp.json`) and `opencode` (built-in MCP support, config in `opencode.jsonc`).
  Use when the user asks to "connect pi/opencode to llm-wiki", "hook up llm wiki", "set up llm-wiki
  MCP", "pi/opencode mcp llm-wiki", "test if llm-wiki is connected", or troubleshoots pi/opencode
  not seeing `llm_wiki_*` tools. Merged from the former `connect-pi-llm-wiki` and
  `connect-opencode-llm-wiki` skills (same MCP server, two different agents).
---

# Connect an agent to the LLM Wiki desktop app via MCP

Wire the LLM Wiki desktop app's local knowledge-base API into an agent as an MCP tool source, so
it can call `llm_wiki_*` tools while still using its own configured LLM provider for reasoning.
LLM Wiki is not an LLM backend itself — its `/chat` endpoint is a bound, non-streaming retrieval
agent scoped to one project, not an OpenAI-compatible completions endpoint. Do not attempt to
register it as a model provider in either agent.

Both agents point at the **same bundled MCP server**:
`%LOCALAPPDATA%\LLM Wiki\mcp-server\dist\src\index.js` (reads `LLM_WIKI_API_TOKEN` and
`LLM_WIKI_API_BASE_URL` from env — see `mcp-server/dist/src/api-client.js`). Only the wiring
mechanism differs per agent — pick the section below.

---

## For `pi` (`@earendil-works/pi-coding-agent`)

### Key facts

- pi's core has **no built-in MCP client** ("No MCP" is a documented design choice in the
  installed package's README). A bare `~/.config/mcp/mcp.json` does nothing until an MCP-capable
  extension is installed.
- The `pi-mcp-adapter` extension (`npm:pi-mcp-adapter`) adds MCP client support and is what
  actually reads `~/.config/mcp/mcp.json`.
- The MCP adapter prefixes tool names with the server key, so a server named `llm-wiki` exposes
  tools like `llm_wiki_llm_wiki_status`, `llm_wiki_llm_wiki_projects`, etc.

### Procedure

1. Confirm the LLM Wiki desktop app is running and note its local API port (default `19828`) and
   API token from **Settings → API + MCP** inside the app.

2. Locate the bundled MCP server entry point:
   ```bash
   find "$LOCALAPPDATA/LLM Wiki/mcp-server" -maxdepth 2 -not -path "*/node_modules/*"
   ```
   Its `package.json` shows `main: dist/src/index.js`.

3. Install the MCP adapter extension into pi (one-time, per machine):
   ```bash
   pi install npm:pi-mcp-adapter
   ```

4. Write (or merge into) `~/.config/mcp/mcp.json` — this is the path `pi-mcp-adapter` checks first:
   ```json
   {
     "mcpServers": {
       "llm-wiki": {
         "command": "node",
         "args": [
           "C:\\Users\\<user>\\AppData\\Local\\LLM Wiki\\mcp-server\\dist\\src\\index.js"
         ],
         "env": {
           "LLM_WIKI_API_TOKEN": "<token from Settings -> API + MCP>",
           "LLM_WIKI_API_BASE_URL": "http://127.0.0.1:19828"
         }
       }
     }
   }
   ```
   Use `\\` (double backslash) for the Windows path inside JSON.

5. Sanity-check the MCP server itself starts cleanly before touching pi:
   ```bash
   cd "$LOCALAPPDATA/LLM Wiki/mcp-server" && \
   LLM_WIKI_API_TOKEN="<token>" LLM_WIKI_API_BASE_URL="http://127.0.0.1:19828" \
   timeout 3 node dist/src/index.js < /dev/null
   ```
   Expect a single line: `LLM Wiki MCP server v<x.y.z> connected to http://127.0.0.1:19828`.

6. Test the pi <-> llm-wiki connection end-to-end, non-interactively:
   ```bash
   pi -p --no-session "Call the llm_wiki_status MCP tool and show me its raw output."
   ```
   A working connection returns JSON containing `"ok": true` and the current project. If pi
   instead says it has no MCP tools, re-check step 3 (adapter not installed) or step 4 (config
   path/JSON typo).

### Known non-issue

`pi-mcp-adapter`'s `/mcp-auth`-style panel may print:
```
No OAuth-capable MCP servers are configured.
```
This is expected and harmless — it only means none of the configured servers use an OAuth flow.
LLM Wiki authenticates with a static bearer token via `LLM_WIKI_API_TOKEN`, not OAuth, so it will
never appear in that OAuth-server list. It does not indicate a broken or dropped connection;
verify connectivity with the `pi -p` test in step 6 instead.

---

## For `opencode`

### Key facts

- opencode's config schema (`https://opencode.ai/config.json`) defines a `McpLocalConfig` type:
  required `type: "local"` + `command` (array of strings), optional `cwd`, `environment` (object
  of string env vars), `enabled` (bool), `timeout` (ms, default 5000). No other keys allowed
  (`additionalProperties: false`).
- Unlike `pi`, opencode has MCP support **built in** — no adapter extension needed, just a `mcp`
  block in `C:\Users\HCH\.config\opencode\opencode.jsonc`.
- The token is already generated once for the `pi` MCP setup and lives in
  `~/.config/mcp/mcp.json` under `mcpServers.llm-wiki.env` — reuse it instead of generating a new
  one from the app's Settings → API + MCP page, unless it's been rotated.

### Procedure

1. Confirm the LLM Wiki desktop app is running (default API port `19828`).
2. Get the token: read `~/.config/mcp/mcp.json` → `mcpServers.llm-wiki.env.LLM_WIKI_API_TOKEN`
   (see the pi section above).
3. Merge into `opencode.jsonc` (see `opencode-config` skill for the file's other sections —
   `provider`/`agent`/`mcp` all coexist in this one file):
   ```jsonc
   "mcp": {
     "llm-wiki": {
       "type": "local",
       "command": [
         "node",
         "C:\\Users\\HCH\\AppData\\Local\\LLM Wiki\\mcp-server\\dist\\src\\index.js"
       ],
       "environment": {
         "LLM_WIKI_API_TOKEN": "<token from step 2>",
         "LLM_WIKI_API_BASE_URL": "http://127.0.0.1:19828"
       },
       "enabled": true
     }
   }
   ```
   (Use `\\` for the Windows path inside JSON, same as the pi config.)

### Verifying — do NOT run the desktop exe to check

There is no `opencode` CLI on this machine's PATH — only `D:\APP\opencode-desktop-win-x64.exe`, a
windowed Electron-style app. Running it with CLI-style args
(`opencode-desktop-win-x64.exe mcp list`) does not print to stdout — it silently launches the full
GUI (multiple `OpenCode` processes) instead. If that happens, close it with:
```powershell
Get-Process | Where-Object { $_.ProcessName -eq "OpenCode" } | Stop-Process -Confirm:$false
```
Instead, verify by asking the user to open opencode normally (CLI on another machine, or the
desktop app) and run a request that calls an `llm_wiki_*` tool — or if a real `opencode` CLI is
found on PATH later, use `opencode mcp list` / `opencode run "..."` per the `opencode-config`
skill.

---

## Related

- `aipower-docker-local` skill covers connecting `pi` running **inside a Linux container**
  (cross-host variant, needs different networking than the plain-Windows-host procedure above).
- `tauri-wiki-app` skill covers the LLM Wiki desktop app itself (REST API, OCR, knowledge graph).
- `opencode-config` skill covers `opencode.jsonc`'s other sections.

---

## Conformance Addendum

## When to Use
Wire the LLM Wiki desktop app's local knowledge-base API into an AI coding agent as an MCP tool source — covers both `pi` (needs the `pi-mcp-adapter` extension, config at `~/.config/mcp/mcp.json`) and `opencode` (built-in MCP support, config in `opencode.jsonc`). Use when the user asks to "connect pi/opencode to llm-wiki", "hook up llm wiki", "set up llm-wiki MCP", "pi/opencode mcp llm-wiki", "test if llm-wiki is connected", or troubleshoots pi/opencode not seeing `llm_wiki_*` tools. Merged from the former `connect-pi-llm-wiki` and `connect-opencode-llm-wiki` skills (same MCP server, two different agents).

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
