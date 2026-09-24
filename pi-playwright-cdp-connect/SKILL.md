---
name: pi-playwright-cdp-connect
description: 在 pi agent 中，將 mcp.json 裡設定的 playwright MCP server 接管到「已開啟」的 Chrome/Edge 遠端偵錯 port 9222，操作使用者現有的瀏覽器視窗（保留登入狀態），而不是每次開新的無痕視窗。當使用者要求「用 playwright 操作我現在開著的瀏覽器」、「接管瀏覽器」、「控制我已開的網頁」時使用。
---

# pi Playwright CDP 接管

pi agent 的 `~/.pi/agent/mcp.json` 已將 `playwright` server 設定為固定用 CDP 連到
`http://127.0.0.1:9222`：

```json
"playwright": {
  "command": "npx",
  "args": [
    "-y",
    "@playwright/mcp@latest",
    "--cdp-endpoint",
    "http://127.0.0.1:9222"
  ]
}
```

**鐵則：本 skill 只接管「已經開著」、且監聽 9222 的瀏覽器，絕不自動啟動新的瀏覽器程序。**
若 port 9222 未開放，把下面的啟動指令交給使用者自己執行，不要用 `Start-Process`/`Bash` 幫他開。

## 1. 檢查 port 9222 是否已開放

```powershell
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
  Where-Object { $_.LocalPort -eq 9222 }
```

或用 Git Bash：

```bash
netstat -ano | grep 9222
```

- 有輸出 → 已開放，直接跳到步驟 3。
- 沒輸出 → 交給使用者下面的指令，停在這裡等使用者開好。

## 2. 使用者自行啟動 debug 瀏覽器（僅在 9222 未開放時提供）

Chrome：

```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="C:\Temp\ChromeDebug"
```

Edge：

```powershell
& "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="C:\Temp\EdgeDebug"
```

**重要限制（Chrome 136+ / 新版 Edge 同理）：**
- 預設 profile 無法遠端偵錯，必須用獨立 `--user-data-dir`，登入狀態會保存在這個獨立 profile 裡（第一次要重新登入一次，之後常駐）。
- debug port 只能綁 `127.0.0.1`。

## 3. 讓 pi 載入 playwright MCP server

`mcp.json` 的變更只在**啟動新 session** 時載入。若剛改過設定或這是第一次使用：
1. 確認 `~/.pi/agent/mcp.json` 內有 `playwright` server 設定（見上方）。
2. 重新啟動 pi / 開新 session。
3. 用 `mcp` 工具確認：`mcp({})` 應能看到 `playwright` server 已連線。

## 4. 驗證接管成功

呼叫 playwright 的 tab 列表或截圖工具（實際工具名稱以 `mcp({ server: "playwright" })` 查到的為準，
常見為 `browser_tabs` / `browser_snapshot` / `browser_take_screenshot` 之類）：

```
mcp({ server: "playwright" })          // 列出可用工具
mcp({ tool: "browser_tabs", args: {} }) // 列出目前分頁，確認連到的是使用者的瀏覽器
```

若回報 target/page closed，重新呼叫一次通常就會重新附著。

## 常見坑

### 坑一：其他工具也搶佔 9222
若環境中還有別的自動化腳本也用同一個 port 開自己的瀏覽器，會互踢（對方常見作法是先
`taskkill /F /IM chrome.exe` 全部殺掉再重開）。CDP 支援多客戶端連同一個瀏覽器，正確共存方式：
- 讓對方先啟動、穩定運作
- 這邊接管後開新分頁操作，**不要關掉最後一個分頁**（關掉唯一分頁可能連整個瀏覽器程序都關掉）

### 坑二：CDP 接管會讓原生對話框對「真人」隱形
只要有 CDP 客戶端連著，瀏覽器原生的 `beforeunload`/`alert`/`confirm` 對話框會被導向 CDP 端處理，
使用者畫面上看不到彈窗（會覺得「卡住」）。需要處理時要用對應的 dialog 處理工具
（如 `browser_handle_dialog`），而不是讓使用者自己點。

### 坑三：CDP/Playwright 無法操作瀏覽器「視窗層級」原生 UI
分頁列、右上角關閉鈕、系統選單等不在網頁 DOM 裡，Playwright 座標系統送不到那裡。
需要視窗層級操作（例如 Alt+F4）要用 PowerShell 的 Win32 API/`SendKeys`，不是 playwright MCP 的職責。

### 坑四：mcp.json 修改後沒生效
`~/.pi/agent/mcp.json` 是 pi agent 的使用者層級設定，**改動不會即時套用到目前已開啟的 session**，
必須重開 pi 才會重新讀取並連上新加入/修改的 server。

## 關閉 debug 瀏覽器（使用者要求時）

只關閉佔用 9222 的那個瀏覽器程序，不動使用者其他視窗：

```powershell
$port = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
  Where-Object { $_.LocalPort -eq 9222 }
if ($port) {
    Stop-Process -Id $port.OwningProcess -Force -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 2
}
$check = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
  Where-Object { $_.LocalPort -eq 9222 }
if ($check) { "仍在運行" } else { "已關閉，port 9222 已釋放" }
```

若使用者要求「先登出再關閉」，務必先在網頁內完成登出流程，再執行關閉指令。

---

## Conformance Addendum

## When to Use
在 pi agent 中，將 mcp.json 裡設定的 playwright MCP server 接管到「已開啟」的 Chrome/Edge 遠端偵錯 port 9222，操作使用者現有的瀏覽器視窗（保留登入狀態），而不是每次開新的無痕視窗。當使用者要求「用 playwright 操作我現在開著的瀏覽器」、「接管瀏覽器」、「控制我已開的網頁」時使用。

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
