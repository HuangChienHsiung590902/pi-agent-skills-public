---
name: connect-chrome
description: 將 Playwright MCP 接管到「已開啟」的 Chrome/Edge 遠端偵錯 port 9222，操作使用者現有瀏覽器（保留登入狀態）。當使用者要求「接管 Chrome」、「控制我的瀏覽器」、「操作已開啟的網頁」時使用。只接管現有的 Chrome，絕不自動啟動新的 Chrome——若 port 9222 未開放，請使用者自己用 debug 捷徑開啟。涵蓋 Claude Code（playwright-vrs，`~/.claude.json`）與 pi agent（playwright，`~/.pi/agent/mcp.json`）兩種 host 的設定；兩者接管方式與踩坑完全相同，只差 MCP 設定檔位置。
---

接管 Playwright MCP 到「已經開著」的 Chrome 遠端偵錯 port (9222)。
**鐵則：本 skill 只接管現有 Chrome，絕不開新的 Chrome（PowerShell 不准 Start-Process，Playwright 也不准自己 launch）。**

## 架構（2026-07-08 起，2026-07-24 簡化）

VRS 這邊固定用 `~/.claude.json` 的 `mcpServers.playwright-vrs`（`--cdp-endpoint http://127.0.0.1:9222`）。

> ⚠️ **2026-07-24 更新：舊版 aipower 專用的 `playwright-aipower` MCP server（目標是已不存在的
> `dev-aipower` skill 啟動的 `C:\com\chainsea`）已從 `~/.claude.json` 移除，相關 skill 也已刪除**
> ——`C:\com\chainsea` 這套安裝本身已不存在（目前live的是 `C:\Lab2\chainsea`，見 `ecp-lab2-instance`
> skill）。目前只剩 `playwright-vrs`（固定接 **9222**）這一個固定的 Playwright MCP server，工具
> 名稱用裸的 `browser_*` 或 `mcp__playwright-vrs__browser_*` 都行，不再需要用
> `select:mcp__playwright-vrs__browser_tabs` 這種精確寫法避開兩個同名 server 互撞。
> 若之後又有其他 aipower 實例需要獨立的 Playwright server，新建一個當前實例專用的 server key
> 並同步更新本 skill，不要直接沿用舊的 `playwright-aipower` 設定。

plugin 市集裡原本那個共用的 `playwright@claude-plugins-official` 已停用（`settings.json` 的
`enabledPlugins` 設為 `false`），避免它殘留的 `.mcp.json`／不確定的 port 造成第三個模糊選項。

> **pi agent 版（原 `pi-playwright-cdp-connect` skill，已併入此處）：** 在 pi 裡不是用
> `~/.claude.json`，而是 `~/.pi/agent/mcp.json` 的 `playwright` server（一樣固定 CDP 到 9222）：
> ```json
> "playwright": {
>   "command": "npx",
>   "args": ["-y", "@playwright/mcp@latest", "--cdp-endpoint", "http://127.0.0.1:9222"]
> }
> ```
> pi 也支援 Edge（debug 捷徑改成 `msedge.exe --remote-debugging-port=9222 --user-data-dir="C:\Temp\EdgeDebug"`）。
> `mcp.json` 的變更只在**啟動新 session** 時載入，改完要重開 pi；用 `mcp({})` 確認 `playwright` server 已連線，
> 再用 `mcp({ tool: "browser_tabs", args: {} })` 驗證接管的是使用者的瀏覽器。以下步驟 1–3 的檢查、驗證、
> 關閉邏輯與四個踩坑，Claude 與 pi 兩邊完全通用。

若 `mcp__playwright-vrs__*` 完全搜尋不到（不是連線失敗，是工具不存在），先確認：
1. `~/.claude.json` 的 `mcpServers` 裡有沒有 `playwright-vrs` 這個 key
2. 新增/修改 `mcpServers` 屬於使用者層級設定，**不是**靠 `/reload-plugins` 生效，需要重新啟動 Claude Code

### 1. 檢查 port 9222 是否已開放

```powershell
$conn = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue | Where-Object { $_.LocalPort -eq 9222 }

if ($conn) {
    Write-Host "✓ Port 9222 已開放，準備接管現有 Chrome"
} else {
    Write-Host "✗ Port 9222 未開放。本 skill 不會自動開新 Chrome。"
    Write-Host "  請自己用 debug 捷徑開啟 Chrome 後再執行 /connect-chrome："
    Write-Host '  "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir="C:\Temp\ChromeDebug"'
}
```

若 port 9222 未開放，**就停在這裡**，把上面那行 debug 捷徑指令交給使用者，請他自己開啟 Chrome，不要呼叫 `Start-Process` 或任何方式自動啟動 Chrome。

### 2. 驗證連線

先用 `mcp__playwright-vrs__browser_tabs` (action: list) 列出分頁；若回報 "Target page... closed"，再呼叫一次即可重新附著。
接著用 `mcp__playwright-vrs__browser_take_screenshot` 截圖確認已成功接管 Chrome。

### 3. 關閉 Chrome（當使用者要求「關閉 chrome」、「退出瀏覽器」時）

只關閉 debug Chrome（占用 port 9222 的那個程序），不影響其他 Chrome：

```powershell
$port = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue | Where-Object { $_.LocalPort -eq 9222 }
if ($port) {
    Stop-Process -Id $port.OwningProcess -Force -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 2
}
$check = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue | Where-Object { $_.LocalPort -eq 9222 }
if ($check) { "✗ Chrome 仍在運行" } else { "✓ Debug Chrome 已關閉，port 9222 已釋放" }
```

> 若使用者同時要求「先登出系統再關閉」，務必**先**在網頁內完成登出流程，**再**執行上面的關閉指令。

## 三個實測踩到的坑（2026-07-02, 2026-07-05）

### 坑一：別的程式也開 `--remote-debugging-port=9222` 會互踢

如果環境裡還有其他工具（例如某個監控/自動化腳本）也用同一個 port 9222 啟動自己的 Chrome，兩邊會打架：
- 對方啟動時通常會先 `taskkill /F /IM chrome.exe` 把所有 Chrome（含我接管的）砍光，再開自己的
- 我這邊如果把**唯一剩下的分頁**關掉（`browser_tabs close` 關到最後一個），會連整個 Chrome 程序一起關掉，一樣會把對方的連線也砍斷
- 結果就是雙方互相斷線、重連，看起來像「不穩定」，其實是兩個工具搶同一個瀏覽器程序

CDP 本身**支援多客戶端同時連同一個瀏覽器**，不是不能共用。正確共存方式：
1. 讓對方的程式先啟動、穩定運作（它做完自己的 `taskkill`+重開）
2. 我後接管，**開新分頁操作，不要關掉最後一個分頁**
3. 需要關閉時只關自己開的分頁，留至少一個分頁活著

### 坑二：CDP 接管會讓瀏覽器原生對話框對「真人」隱形

只要有 CDP 除錯客戶端（不管是我的 Playwright MCP 還是任何自動化工具）連著，瀏覽器的原生對話框（`beforeunload` 離開確認、`alert`/`confirm` 等）會被**攔截給 CDP 客戶端處理**，不會渲染顯示在使用者實際看到的螢幕上——使用者會看到「視窗卡住、什麼都沒跳出來、關不掉」，但其實是對話框存在、只是只有 CDP 那端看得到（用 `browser_handle_dialog` 才能處理）。

**這代表：只要我接管著 Chrome，就沒辦法讓使用者乾淨地測試「手動操作瀏覽器原生 UI 會發生什麼」這件事**（例如驗證 `beforeunload` 警告框會不會跳出來）。這類測試要嘛請使用者在我斷開連線之後自己測，要嘛接受這個測試本身就會被我的接管干擾、結果不能代表真實使用者的體驗。

也順帶確認了 CDP/Playwright 這類自動化工具**完全無法**點擊瀏覽器視窗本身的原生 UI（分頁列、右上角關閉鈕、系統選單）——那些東西不在網頁 DOM 裡，滑鼠點擊工具的座標系統再怎麼設都送不到那裡，這是協議層的邊界，不是工具設定問題。要做到「視窗層級」的操作（例如用 Alt+F4 模擬真人關閉），得跳出 Playwright，改用 PowerShell 的 Win32 API／`SendKeys` 直接對視窗下手。

### 坑三：plugin 沒在 enabledPlugins 啟用，症狀跟「cdp-endpoint 沒設定」一模一樣

`.mcp.json` 設定正確、`/reload-plugins` 跑了好幾次，`browser_*` 工具還是完全搜尋不到——花了一輪才發現是 `C:\Users\HCH\.claude\settings.json` 的 `enabledPlugins` 裡根本沒有 `playwright@claude-plugins-official` 這個 key。plugin 檔案存在於 marketplace 目錄不代表它被啟用；沒啟用就沒有 MCP server 進程，工具自然不存在。這跟步驟 0 的 cdp-endpoint 檢查是兩個獨立的失敗點，必須都檢查（見上方新增的步驟 0.5）。

### 坑四：Bash 工具的 `cmd //c "xxx.bat"` 會誤報「不是內部或外部命令」

用 Bash 工具（Git Bash）執行 `cmd //c "some.bat"`（即使 cwd 正確、`dir some.bat` 也能找到檔案，甚至連一個
全新、內容只有 `echo hello` 的 `.bat` 都一樣）會報：
```
'some.bat' is not recognized as an internal or external command, operable program or batch file.
```
這是 Bash 工具呼叫 `cmd //c` 時的參數/quoting 問題，**不是 .bat 檔案本身的錯**，也跟 PATHEXT、檔案權限、
cwd 都無關（`cmd //c "cd"`、`cmd //c "dir xxx.bat"` 這類「內建命令+參數」的呼叫是正常的，只有「直接執行
.bat 檔名」這種呼叫會壞）。

**解法：改用 PowerShell 工具的 `Start-Process`**，一律可靠：
```powershell
Start-Process -FilePath 'cmd.exe' -ArgumentList '/c', 'C:\full\path\to\some.bat' -WorkingDirectory 'C:\full\path' -WindowStyle Minimized
```
需要看輸出時再接 `>` 重導向到檔案（重導向字串也放進同一個 `-ArgumentList` 字串裡，不要拆成太多零碎片段，
否則同樣可能因為 quoting 兜不起來而失敗）：
```powershell
Start-Process -FilePath 'cmd.exe' -ArgumentList '/c', 'C:\full\path\to\some.bat > C:\log.txt 2>&1' -WorkingDirectory 'C:\full\path'
```

---

**說明：**
- 本 skill **只接管**已開著的 Chrome（使用者用 debug 捷徑開的），保留登入狀態
- port 9222 未開放時，**不自動開新 Chrome**，改為提示使用者自己開
- 每次重啟 Chrome 後需執行 `/reload-plugins` 重新連線

**重要限制（Chrome 136+）：**
- 預設 profile **無法**遠端偵錯，必須用獨立 `--user-data-dir`（這裡用 `C:\Temp\ChromeDebug`），登入狀態永久保留在此 profile
- debug port 只能綁 `127.0.0.1`，`--remote-debugging-address` 已無效，不需要寫
- 使用者的 debug 捷徑目標：`"...\chrome.exe" --remote-debugging-port=9222 --user-data-dir="C:\Temp\ChromeDebug"`

---

## Conformance Addendum

## When to Use
將 Playwright MCP 接管到「已開啟」的 Chrome/Edge 遠端偵錯 port 9222，操作使用者現有瀏覽器（保留登入狀態）。當使用者要求「接管 Chrome」、「控制我的瀏覽器」、「操作已開啟的網頁」時使用。只接管現有的 Chrome，絕不自動啟動新的 Chrome——若 port 9222 未開放，請使用者自己用 debug 捷徑開啟。涵蓋 Claude Code（playwright-vrs，`~/.claude.json`）與 pi agent（playwright，`~/.pi/agent/mcp.json`）兩種 host 的設定；兩者接管方式與踩坑完全相同，只差 MCP 設定檔位置。

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
