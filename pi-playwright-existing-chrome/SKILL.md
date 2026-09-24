---
name: pi-playwright-existing-chrome
description: 透過 Playwright Extension 與 Playwright MCP 接管使用者目前已開啟的正常 Chrome，保留既有分頁、登入狀態與 cookies。當使用者要求操作原有 Chrome、查詢網站、使用目前 Chrome 的登入狀態、不要新開瀏覽器，或明確說「接管現有 Chrome」時使用；不要改用獨立 profile 或 9222 CDP。
---

# Playwright 接管現有 Chrome

## 目的

使用官方 Playwright Extension，讓 Pi 透過 Playwright MCP 操作使用者日常 Chrome 中被授權的分頁。這個流程不是啟動新的 Chrome，也不是使用 `C:\Temp\ChromeDebug` 或 `127.0.0.1:9222`。

目前 Administrator profile 的共用 MCP 設定是：

```text
C:\Users\Administrator\.config\mcp\mcp.json
```

Playwright server 應使用：

```json
{
  "playwright": {
    "command": "C:/Program Files/nodejs/npx.cmd",
    "args": [
      "-y",
      "@playwright/mcp@latest",
      "--extension",
      "--profile-dir-name",
      "Default"
    ],
    "lifecycle": "lazy"
  }
}
```

## 事前設定

1. 正常開啟使用者平常的 Chrome。
2. 確認 Chrome 已安裝並啟用官方 **Playwright Extension**：
   ```text
   https://chromewebstore.google.com/detail/playwright-extension/mmlmfjhmonkocbjadbfplnigmagldckm
   ```
3. 若 Playwright MCP 設定剛變更，請要求使用者在 Pi 執行 `/reload`；若仍未更新，重新啟動 Pi。
4. 首次連接時，Chrome 會顯示 Playwright Extension 的連接／分頁選擇頁。使用者需要批准連接，並選擇允許 Pi 操作的分頁。

## 操作流程

1. 先確認 MCP 狀態與 Playwright server 是否可用：
   ```text
   mcp({})
   mcp({ server: "playwright" })
   ```
2. 首次連接時，若目前分頁是 `chrome-extension://.../connect.html`，確認頁面顯示 `pi-mcp-playwright connected`。若尚未授權，停止並請使用者在 Chrome 完成批准／選擇分頁。
3. 使用 Playwright 工具列出目前可操作的分頁：
   ```text
   mcp({ tool: "playwright_browser_tabs", args: { action: "list" } })
   ```
4. 只對使用者授權的現有分頁進行操作。查資料時優先使用 `navigate`、`snapshot`、`find`、`evaluate` 與 `tabs`；需要互動時才使用 `click`、`fill` 或 `press_key`。
5. 操作前先取得最新 snapshot，不要沿用過期的 element ref。完成後回報實際使用的分頁與資料時間。

## 重要限制

- 不要執行 `Chrome-CDP-9222.exe`，也不要把 Playwright MCP 改回 `--cdp-endpoint http://127.0.0.1:9222`。
- `--extension` 只連接已安裝擴充功能的 Chrome profile；`--profile-dir-name Default` 對應 `chrome://version` 顯示的 Profile Path 最後一段。若使用者改用其他 profile，應改成該 profile directory name。
- 連接頁面可能會先出現一個新的 extension tab；這是連線授權頁，不代表啟動了新的 Chrome。授權後應列出並選取使用者原本的網頁分頁。
- Playwright 可以操作網頁 DOM 與分頁內容，但不能直接操作 Chrome 網址列、工具列、擴充功能選單或其他瀏覽器原生 UI。
- `chrome://`、Chrome Web Store 某些頁面、瀏覽器內部頁面及未授權分頁可能無法操作。
- 登入狀態與 cookies 屬於使用者敏感資料；不要輸出 cookies、token、密碼或完整的私人頁面內容。需要登入時讓使用者自行完成。
- 執行送出、刪除、付款、發文、下載或其他有副作用的操作前，先確認目標與使用者意圖。

## 故障排除

### MCP server 未連線

1. 確認設定檔存在且 JSON 有效：
   ```powershell
   $p = "$env:USERPROFILE\.config\mcp\mcp.json"
   Get-Content -Raw $p | ConvertFrom-Json | Out-Null
   ```
2. 確認 `playwright` 的 args 包含 `--extension`，且沒有 `--cdp-endpoint`。
3. 執行 `/reload`；仍失敗則重新啟動 Pi。

### 只看到 Welcome／connect.html

這通常代表 MCP server 已連到擴充功能，但尚未選定要授權的網頁分頁。請使用者在 Chrome 的 Playwright Extension 連線頁完成批准和分頁選擇，再重新呼叫 `playwright_browser_tabs`。

### 看不到使用者原本的分頁

- 確認擴充功能安裝在正確的 Chrome profile。
- 在該 profile 開啟 `chrome://version`，核對 Profile Path 最後一段；必要時修正 `--profile-dir-name`。
- 確認使用者已把目標分頁加入 Playwright Extension 的 tab group／授權範圍。
- 不要為了排查而關閉使用者全部 Chrome；先要求使用者確認擴充功能狀態。

## 驗證

成功接管的證據至少包括：

1. `mcp({ server: "playwright" })` 可列出 Playwright 工具。
2. `playwright_browser_tabs` 列出使用者原本 Chrome 的實際網頁分頁，而不只是 extension connect 頁。
3. 對其中一個已授權分頁執行 `playwright_browser_snapshot`，取得正確的頁面標題、URL 與 DOM 內容。
4. 回報時明確說明是透過正常 Chrome 的 Playwright Extension 接管，不是新開獨立瀏覽器。
