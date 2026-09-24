---
name: webmcp-public-token-cloudflare-bridge
description: Fix and maintain the local @jason.today/webmcp MCP token flow for the public LINE-protected WebMCP demo at https://webmcp.james-huang.org, including Cloudflare websocket bridge, ws://localhost token rewrite, localStorage persistence across refresh/page navigation, tokenless UX, and connection status synchronization.
triggers:
  - webmcp token
  - /webmcp
  - webmcp public token
  - webmcp-ws.james-huang.org
  - ws://localhost:4797
  - WebMCP 不能連
  - Invalid LINE login state
  - WebMCP 換頁打勾消失
  - WebMCP refresh 後未連線
  - 不要手動貼 token
  - 右下角打勾但顯示尚未連線
---

# WebMCP Public Token Cloudflare Bridge

## When to Use

Use this skill when the user mentions any of these:

- `/webmcp` 產生的 token 仍是 `ws://localhost:4797`
- public site `https://webmcp.james-huang.org/connect` cannot connect after pasting a WebMCP token
- WebMCP token must work from a public HTTPS/LINE Login page rather than only from local browser
- `webmcp-ws.james-huang.org`, `localhost:4797`, `Cloudflare Tunnel`, or SSH reverse tunnel for WebMCP
- LINE Login callback state error for this WebMCP demo (`Invalid LINE login state.`)
- right-bottom WebMCP checkmark disappears after refresh/page navigation
- `/connect` says `尚未連線` even though the WebMCP widget has a `✓`
- user asks to avoid manually pasting WebMCP tokens; prefer agent/CDP/deep-link/localStorage auto-pairing

Do **not** use this for generic MCP server development unless the task specifically involves this local `@jason.today/webmcp` deployment and public demo site.

## Current Architecture

- Local WebMCP MCP server runs on this Windows machine:
  - process: `node D:\MCP\webmcp-mcp\launcher.cjs --mcp --forked`
  - local server: `ws://localhost:4797`
- Public WebMCP demo site runs on Jetson:
  - SSH: `ssh hch@10.145.119.12`
  - app folder: `/home/hch/webmcp-demo-line-auth`
  - container: `webmcp-demo`
  - host port: `18082 -> container 80`
  - public site: `https://webmcp.james-huang.org`
- Cloudflare Tunnel on Jetson:
  - tunnel name: `webmcp`
  - config: `/etc/cloudflared/webmcp.yml`
  - service: `cloudflared-webmcp`
  - site route: `webmcp.james-huang.org -> http://127.0.0.1:18082`
  - websocket route: `webmcp-ws.james-huang.org -> http://127.0.0.1:14797`
- SSH reverse tunnel from Windows to Jetson:
  - `Jetson 127.0.0.1:14797 -> Windows 127.0.0.1:4797`
  - command:
    ```powershell
    ssh -N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -R 127.0.0.1:14797:127.0.0.1:4797 hch@10.145.119.12
    ```

## Key Rule

For the public HTTPS site, token JSON must contain:

```json
{"server":"wss://webmcp-ws.james-huang.org","token":"<actual-token>"}
```

A token containing this is only local-machine usable and will fail from the public site / phone / LINE browser:

```json
{"server":"ws://localhost:4797","token":"<actual-token>"}
```

## Procedure

### 1. Decode a token to inspect the server

```powershell
$token = '<base64 token from /webmcp>'
$json = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($token))
$json
```

Or with Python:

```bash
python - <<'PY'
import base64,json
s='<base64 token>'
print(json.dumps(json.loads(base64.b64decode(s + '===')), indent=2))
PY
```

If `server` is `ws://localhost:4797`, the public site cannot use it directly.

### 2. Quick workaround: rewrite only the token envelope

When the WebMCP session token value is still valid, rewrite the same `token` value with the public server:

```python
import base64, json
old = '<base64 token from /webmcp>'
data = json.loads(base64.b64decode(old + '==='))
data['server'] = 'wss://webmcp-ws.james-huang.org'
print(base64.b64encode(json.dumps(data, separators=(',', ':')).encode()).decode())
```

Paste the printed token into `https://webmcp.james-huang.org/connect`.

### 3. Permanent fix: patch token generation

The launcher is:

```text
D:\MCP\webmcp-mcp\launcher.cjs
```

It should set:

```js
process.env.WEBMCP_PUBLIC_SERVER = process.env.WEBMCP_PUBLIC_SERVER || "wss://webmcp-ws.james-huang.org";
```

The installed upstream bundle currently generates tokens in:

```text
D:\.system\.mcp\npm\node_modules\@jason.today\webmcp\build\index.js
```

Patch the generated-token object so it uses `process.env.WEBMCP_PUBLIC_SERVER` if present. The minified snippet should effectively be:

```js
s = { server: process.env.WEBMCP_PUBLIC_SERVER || `ws://${e}`, token: t }
```

instead of always:

```js
s = { server: `ws://${e}`, token: t }
```

After patching, validate syntax:

```bash
node -c D:/MCP/webmcp-mcp/launcher.cjs
node -c D:/.system/.mcp/npm/node_modules/@jason.today/webmcp/build/index.js
```

Important: the already-running WebMCP MCP server may still use old code. Restart the MCP server / Pi session before expecting `/webmcp` to emit public tokens.

### 4. Ensure the SSH reverse tunnel is running

Check local WebMCP server:

```powershell
Get-NetTCPConnection -LocalPort 4797 -ErrorAction SilentlyContinue
Invoke-WebRequest -UseBasicParsing -TimeoutSec 3 http://127.0.0.1:4797
```

Start or restart reverse tunnel from Windows:

```powershell
$pattern = '14797:127.0.0.1:4797'
Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -like "*$pattern*" -and $_.Name -like 'ssh*' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }

$argsList = '-N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -R 127.0.0.1:14797:127.0.0.1:4797 hch@10.145.119.12'
Start-Process -FilePath 'ssh.exe' -ArgumentList $argsList -WindowStyle Hidden
```

Verify from Jetson:

```bash
ssh hch@10.145.119.12 'ss -ltnp | grep 14797 || true; curl -fsSI --max-time 5 http://127.0.0.1:14797/; curl -fsSI --max-time 10 https://webmcp-ws.james-huang.org/'
```

### 5. Ensure Cloudflare websocket route exists

On Jetson, `/etc/cloudflared/webmcp.yml` should include:

```yaml
ingress:
  - hostname: webmcp.james-huang.org
    service: http://127.0.0.1:18082
  - hostname: webmcp-ws.james-huang.org
    service: http://127.0.0.1:14797
  - service: http_status:404
```

Restart if changed:

```bash
ssh hch@10.145.119.12 'sudo systemctl restart cloudflared-webmcp && sleep 3 && systemctl is-active cloudflared-webmcp'
```

### 6. Public site must serve `webmcp.js`

If `/connect` shows the UI but Connect does nothing, verify the JS file exists in the container:

```bash
ssh hch@10.145.119.12 'curl -fsSI --max-time 10 https://webmcp.james-huang.org/webmcp.js'
```

If missing, copy it from the original demo source and rebuild:

```bash
ssh hch@10.145.119.12 'set -e
APP=/home/hch/webmcp-demo-line-auth
cp /home/hch/webmcp-demo/site/webmcp.js "$APP/public/webmcp.js"
docker build -t webmcp-demo-line-auth:latest "$APP" >/tmp/webmcp-demo-line-auth-build-webmcpjs.log
# recreate webmcp-demo preserving env secrets; do not print LINE_CHANNEL_SECRET
'
```

Preserve env vars from the old container when recreating it; never print `LINE_CHANNEL_SECRET` in user-visible output.


## WebMCP Persistence / Tokenless UX Fixes

Use this section when the right-bottom WebMCP `✓` disappears after refresh or page navigation, when `/connect` says `尚未連線` despite the widget showing `✓`, or when the user complains about having to manually paste tokens.

### Persist connection in localStorage

The demo is multi-page (`/`, `/products`, `/product/:id`, `/cart`, `/admin`, `/connect`). The connected widget must keep the `✓` after refresh and page navigation.

Patch `public/webmcp.js` so connection metadata uses `localStorage`, not `sessionStorage`:

- storage key: `webmcp_token`
- `token`: original/base64 pairing token
- `originalToken`: same original/base64 token for reconnecting
- `authToken`: registered channel token returned by the WebMCP server
- `server`, `host`, `channel`: routing metadata

Expected stored shape:

```json
{
  "token": "<original-base64-token>",
  "originalToken": "<original-base64-token>",
  "authToken": "<registered-channel-token>",
  "server": "wss://webmcp-ws.james-huang.org",
  "host": "webmcp_james-huang_org",
  "channel": "/webmcp_james-huang_org"
}
```

Implementation rules:

1. `_checkStoredToken()` reads `localStorage.getItem(this.CONNECTION_STORAGE_KEY)`.
2. Reconnect with `connectionInfo.originalToken || connectionInfo.token`, never with the registered `authToken` as if it were base64.
3. If stored metadata for the same `server` and `host` has `authToken`, skip registration and connect directly to `storedInfo.channel` using `storedInfo.authToken`.
4. On `pagehide` / `beforeunload`, do not clear localStorage. Browser WebSocket close code `1001` during navigation is normal.
5. Only clear `localStorage.webmcp_token` on explicit `disconnect()` or true authorization failure (`401`, `4001`, `4401`).
6. Do not clear the token on transient socket errors; let the next page/reload retry.

### Avoid manual token UX

The protocol still needs a credential internally, but the user should not copy/paste it. Prefer:

1. Agent gets a token and injects it through CDP: `window.webmcp.connect(token)`.
2. Agent opens a one-time deep link such as `?webmcp_token=<base64>` or `#webmcp_token=<base64>`; `shop.js` captures it, calls `webmcp.connect(token)`, then removes it from the URL with `history.replaceState()`.
3. Keep `/connect` only as a fallback/debug page.

For normal shop pages, instantiate WebMCP like:

```js
const webmcp = new WebMCP({
  color: "#f54e00",
  position: "bottom-right",
  showTokenInput: false,
  inactivityTimeout: 24 * 60 * 60 * 1000
});
```

If `showTokenInput:false`, hide the token input row and show a neutral hint:

```text
由 Agent 自動連線，不需手動貼 token
```

### Cache bust runtime JS

After changing `webmcp.js` or `shop.js`, add versioned script URLs in every page:

```html
<script src="/webmcp.js?v=localstorage-YYYYMMDD-N"></script>
<script src="/shop.js?v=localstorage-YYYYMMDD-N"></script>
```

Also add no-store routes before `express.static()` in `server.js`:

```js
app.get("/webmcp.js", (req, res) => {
  res.setHeader("Cache-Control", "no-store, max-age=0");
  res.sendFile(path.join(PUBLIC_DIR, "webmcp.js"));
});
app.get("/shop.js", (req, res) => {
  res.setHeader("Cache-Control", "no-store, max-age=0");
  res.sendFile(path.join(PUBLIC_DIR, "shop.js"));
});
```

### Synchronize `/connect` status

If the right-bottom widget shows `✓`, `/connect` must not display `尚未連線`. In `public/connect.html`, poll `webmcp.isConnected`:

```js
const tokenStatus = document.getElementById("webmcp-token-status");
function refreshTokenStatus(){
  const connected = !!webmcp.isConnected;
  tokenStatus.textContent = connected ? "已連線" : "尚未連線";
  tokenStatus.style.color = connected ? "var(--success)" : "var(--muted)";
}
setTimeout(refreshTokenStatus, 300);
setInterval(refreshTokenStatus, 1000);
```

After `webmcp.connect(token)` or `webmcp.disconnect()`, call `refreshTokenStatus()` rather than writing a fixed stale message.

### Deploy after changes

Local working copy is usually `C:/Users/HCH/_webmcp_ec`; remote app folder is `/home/hch/webmcp-demo-line-auth` on `hch@10.145.119.12`.

Backup touched files, copy into `public/` as appropriate, rebuild `webmcp-demo-line-auth:latest`, then recreate `webmcp-demo` preserving environment variables and `-v ~/webmcp-demo-line-auth/data:/app/data`. Never print `LINE_CHANNEL_SECRET` or active WebMCP tokens in final answers.

## WebMCP Tool-Call Failure Fixes (server bundle + demo data)

### 1. Tool call routed to stale socket → "Tool call timed out"

Symptom: tools list fine (13 tools visible) but every tool call returns `Tool call timed out` (30s, from `function jl` / 3e4 in the bundle).

Root cause: the installed bundle picked the first client in a channel:

```js
let d=C[a].values().next().value
```

After refresh/reconnect there can be stale sockets in the channel map, so the `callTool` goes to a dead socket.

Fix (patched in `D:\.system\.mcp\npm\node_modules\@jason.today\webmcp\build\index.js`, backups `*.bak-ready-client-20260828*`): choose the newest OPEN client and fail fast when none exists:

```js
let d=[...C[a]].filter(f=>f&&f.readyState===1).at(-1);
if(!d){t.send(JSON.stringify({id:s,type:"toolResponse",error:`No ready clients available in channel ${a} to handle tool: ${l}`}));return}
```

Same pattern was applied to the prompt-call and resource-call sites (they return `promptResponse` / `resourceResponse` respectively).

### 2. Tool result returned as raw object → "(empty result)"

Symptom: calls no longer time out but Pi shows `(empty result)`.

Root cause: the `tools/call` request handler returned the raw resolved value (a plain JSON object from the website) instead of an MCP `CallToolResult`.

Fix (backup `*.bak-toolresult-content-20260828*`): wrap non-content-array results:

```js
let s=await r;
return s&&typeof s==="object"&&Array.isArray(s.content)?s:{content:[{type:"text",text:typeof s==="string"?s:JSON.stringify(s,null,2)}]}
```

Note: after patching `build/index.js` the running MCP server must be stopped and reconnected (Pi `mcp connect` restarts it) — this invalidates the browser WebSocket pairing, so a fresh token is needed.

### 3. Product data missing `id` → get_product/add_to_cart always fail

The 50 seed products in `/home/hch/webmcp-demo-line-auth/data/products.json` had no `id`, so all id-based tools returned `product not found`. Fixed by assigning ids 1–50 once and adding `normalizeProductIds()` in `server.js` (backup `server.js.bak-normalize-product-ids-*`) that auto-fills missing ids at startup.

### 4. `refreshPageProducts is not defined` in add/update/delete_product

`public/shop.js` used `refreshPageProducts?.()`, which throws `ReferenceError` when the variable is not declared at all (e.g. on `/connect`). The POST still succeeded server-side, so failed tool calls could leave orphan test products in the DB — always check `data/products.json` after a failing write test. Fixed with an explicit guard:

```js
(globalThis.refreshPageProducts && typeof globalThis.refreshPageProducts === "function" ? globalThis.refreshPageProducts() : (console.warn("[webmcp] refreshPageProducts not available"), Promise.resolve()))
```

Note: `node --check` rejects ESM `server.js` (`import` syntax); use `docker run --rm -v "$PWD":/app -w /app node:20-alpine node --check server.js` instead.

### 5. Browser keeps old `shop.js` even after container rebuild

Registered WebMCP tools live in the page's JS. Rebuilding the container does nothing until the user's page actually reloads the new `shop.js` — bump the `?v=` cache-buster in every HTML page (`webmcp.js?v=...`, `shop.js?v=...`) and have the user do Ctrl+F5 + reconnect.

### Verification: all 13 site tools

Read: `get_page_summary`, `list_products` (query + category), `get_product`, `get_cart_summary`, resource `webmcp-shop://products`. Cart: `add_to_cart`, `set_cart_quantity`, `clear_cart`. Write: `add_product`, `update_product`, `delete_product`. Test writes with a 測試-category product and clean it up afterwards (check `/api/products` count returns to baseline).

## LINE Login State Fix

If callback returns:

```text
Invalid LINE login state.
```

Patch `/home/hch/webmcp-demo-line-auth/server.js` so `/login` sends a signed state value that can validate itself, not only a cookie-backed random state. Cookie-only state can be lost in LINE app / mobile cross-domain flows.

Expected behavior after fix: using a fake `code=dummy` should no longer return `Invalid LINE login state`; it should reach LINE token exchange and return an `invalid_grant`-type error for the fake code.

## Pitfalls

- Patching `index.js` does not affect an already-running MCP process. Restart Pi / MCP server before testing `/webmcp` output.
- `wss://webmcp-ws.james-huang.org` works only while the Windows-to-Jetson SSH reverse tunnel is alive.
- `localhost` in browser JavaScript means the device running the browser, not the Windows machine running Pi.
- Do not expose or print `LINE_CHANNEL_SECRET`, `SESSION_SECRET`, or active WebMCP tokens in final answers.
- The public site is HTTPS, so `ws://` may be blocked as mixed content; use `wss://`.
- Browser cache may keep an old `/webmcp.js` or `/connect` page. Use Ctrl+F5 or cache-busting when verifying.
- Do not rely on `sessionStorage` for WebMCP connection persistence. Store reconnect metadata in `localStorage.webmcp_token`.
- A right-bottom `✓` is the source of truth for widget connection; do not show `尚未連線` elsewhere if `webmcp.isConnected === true`.
- Avoid asking the user to manually paste tokens. Token credentials are still needed internally, but the UX should be CDP/deep-link/agent-injected and persisted.

## Verification

1. New `/webmcp` token decodes to:
   ```json
   {"server":"wss://webmcp-ws.james-huang.org","token":"..."}
   ```
2. Local WebMCP server responds:
   ```powershell
   Invoke-WebRequest http://127.0.0.1:4797
   ```
3. Jetson reverse port responds:
   ```bash
   ssh hch@10.145.119.12 'curl -fsSI http://127.0.0.1:14797/'
   ```
4. Public websocket hostname responds over HTTPS:
   ```bash
   curl -fsSI https://webmcp-ws.james-huang.org/
   ```
5. Public site serves widget JS:
   ```bash
   curl -fsSI https://webmcp.james-huang.org/webmcp.js
   ```
6. On `https://webmcp.james-huang.org/connect`, connect and confirm the status changes from `尚未連線` to connected.
7. After connecting, inspect browser state: `window.webmcp.isConnected === true`, right-bottom trigger text is `✓`, `localStorage.webmcp_token` is non-empty, and `sessionStorage.webmcp_token` is empty/null.
8. Navigate from `/` to `/products`, wait a few seconds, and confirm the right-bottom `✓` remains.
9. Refresh the browser and confirm the right-bottom `✓` remains.
10. On `/connect`, if the right-bottom widget has `✓`, the page status must say `已連線`, not `尚未連線`.

## Inputs and Outputs

### Inputs

- 本機 WebMCP server、SSH reverse tunnel、Jetson service 與 public site 的連線狀態。
- `/webmcp` 產生的 token，以及需要驗證的 public URL。

### Outputs

- 可由 public site 使用的 `wss://webmcp-ws.james-huang.org` token flow。
- token、localStorage、Cloudflare/SSH/Jetson 變更的摘要與驗證結果。
- 若無法修復，指出斷線的具體鏈路。

## Rules and Limitations

- 不要把含 credential 的 token 或 Authorization header 寫入公開文件、git 或一般日誌。
- 只使用本機 `4797`、Jetson reverse tunnel 與已授權的 public deployment；不要擅自更換 production domain。
- public HTTPS 頁面必須使用 `wss://`，不能把 `ws://localhost:4797` token 直接交給外部瀏覽器。
- 修改 Cloudflare、SSH tunnel 或遠端容器前先備份設定，完成後必須做端到端驗證。
