---
name: webmcp-quick-fix
description: 一鍵修復 WebMCP token 無法連線的問題。當 /webmcp 產生的 token 貼到網站後無法連線、或 https://webmcp-ws.james-huang.org 回 502、或 WebMCP 右下角無法打勾時使用。
triggers:
  - WebMCP token 不能用
  - webmcp token 連不上
  - webmcp-ws 502
  - /webmcp 連線失敗
  - WebMCP 不能連
  - token 為啥不能用
  - webmcp 修復
  - webmcp quick fix
---

# WebMCP Quick Fix

## When to Use

當使用者說 WebMCP token 貼到網站後無法連線，或 `webmcp-ws.james-huang.org` 回 502/連不上，或右下角 WebMCP widget 無法打勾時。

## 連線架構（一句話）

```
網站 → wss://webmcp-ws.james-huang.org → Cloudflare Tunnel → Jetson:14797 → SSH反向隧道 → 本機:4797 (WebMCP MCP server)
```

三個環節任何一個斷了 token 就不能用。

## 快速修復（三步驟）

### Step 1: 跑自動修復腳本

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "D:\OB\skills\webmcp-quick-fix\scripts\webmcp_quick_fix.ps1"
```

這個腳本會自動：
1. 檢查本機 4797 埠（WebMCP MCP server）
2. 如果沒在聽 → 嘗試 `mcp connect webmcp`
3. 檢查 SSH 反向隧道（本機→Jetson 14797）
4. 如果隧道失效 → 殺掉舊的，重建新的
5. 驗證全鏈：`127.0.0.1:4797` → `Jetson:14797` → `webmcp-ws.james-huang.org`

### Step 2: 如果腳本無法自動修復 MCP server

手動執行：

```
mcp connect webmcp
```

等連上後再跑一次 Step 1 的腳本。

### Step 3: 驗證 token 可用

叫 Pi agent 跑 `/webmcp` 產生一個新 token，解碼確認 `server` 欄位是 `wss://webmcp-ws.james-huang.org`（非 `ws://localhost:4797`）。

## 手動檢查（當腳本不能用時）

### 檢查 1：本機 WebMCP server

```powershell
Get-NetTCPConnection -LocalPort 4797 -ErrorAction SilentlyContinue | ft LocalPort,State
```

如果沒輸出 → server 沒跑，執行 `mcp connect webmcp`

### 檢查 2：SSH 反向隧道

```powershell
# 看有沒有舊隧道
Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -like '*14797:127.0.0.1:4797*' } | ft ProcessId
```

如果沒程序或 tunnel 已死 → 重建：

```powershell
# 先殺掉舊的
Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -like '*14797:127.0.0.1:4797*' } | ForEach-Object { Stop-Process -Id $_.ProcessId -Force }

# 重建
Start-Process -FilePath 'ssh.exe' -ArgumentList '-N','-o','ExitOnForwardFailure=yes','-o','ServerAliveInterval=30','-o','ServerAliveCountMax=3','-R','127.0.0.1:14797:127.0.0.1:4797','hch@10.145.119.12' -WindowStyle Hidden
```

### 檢查 3：全鏈驗證

```bash
# 本機
powershell -NoProfile -Command "Invoke-WebRequest -UseBasicParsing -TimeoutSec 3 http://127.0.0.1:4797 | ft StatusCode"

# Jetson 端
ssh -o BatchMode=yes hch@10.145.119.12 'curl -fsSI --max-time 5 http://127.0.0.1:14797/ | head -1'

# 公開網域
curl -sk -m 10 https://webmcp-ws.james-huang.org/ -o /dev/null -w '%{http_code}'
```

三項都回 200 才算通。

## Pitfalls

- SSH 反向隧道靠背景 `ssh -N -R` 維持，本機重開機或 SSH 斷線後需要重建
- WebMCP MCP server 在 Pi agent 重啟後也需要 `mcp connect webmcp`
- 如果 `webmcp-ws.james-huang.org` 回 502，一定是 SSH 隧道斷了（Jetson 14797 沒在聽）
- Token 裡 `server` 是 `ws://localhost:4797` 的話，從外網/手機無法連 → 需要重取 token

## Verification

1. 腳本輸出 `ALL OK` 或手動三項檢查都回 200
2. `/webmcp` 產生的 token 解碼後 `server` = `wss://webmcp-ws.james-huang.org`
3. 貼到 `https://webmcp.james-huang.org/connect` 後右下角顯示 `✓`

## Inputs and Outputs

### Inputs

- 本機 TCP 4797、SSH reverse tunnel、Jetson 14797 與 public WebMCP websocket endpoint 的狀態。
- 目前 token、public URL 及腳本輸出。

### Outputs

- 修復後的 WebMCP server/tunnel 連線。
- `ALL OK` 或各檢查點的 HTTP/TCP 結果，以及可用的 token server 欄位。

## Procedure

1. 先檢查本機 4797 是否有 WebMCP server，再檢查 SSH reverse tunnel 與 Jetson 14797。
2. 只有在對應服務確實失效時才重建 MCP server 或 tunnel；保留可用連線。
3. 依序驗證 `127.0.0.1:4797`、Jetson `127.0.0.1:14797`、public websocket hostname。
4. 產生新 token，確認其 `server` 是 `wss://webmcp-ws.james-huang.org`，最後在 `/connect` 驗證 widget 狀態。

## Rules and Limitations

- 不要把含敏感資料的 token 貼到公開日誌、Issue 或 Skill 文件。
- 不要無條件殺掉所有 SSH 或 node process；先用 command line 與 port 精確識別目標。
- 不要把 `ws://localhost:4797` 當成手機或外部瀏覽器可用的 endpoint。
- 修改 tunnel、Cloudflare 或遠端容器前要保留設定備份；無 SSH 或遠端授權時只能回報檢查結果，不能假裝已修復。
