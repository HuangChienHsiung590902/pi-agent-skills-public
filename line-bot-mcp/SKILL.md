---
name: line-bot-mcp
description: 在這台 Windows 機器安裝、修補、驗證與排查 LINE 官方 @line/line-bot-mcp-server（pi MCP，設定在 ~/.config/mcp/mcp.json）。用在：使用者要裝 LINE Bot MCP、查訊息配額、推文字/Flex、廣播、Rich Menu、get_follower_ids 403、create_rich_menu Windows 失敗、npx 壞掉、或重啟後 MCP 看不到 line-bot。
description_zh: 本機 LINE Bot MCP 的安裝、Windows 修補與驗證流程；含配額查詢、Rich Menu、follower API 帳號限制。
---

# LINE Bot MCP（pi）

## When to Use

當使用者要：

- 安裝、重裝、驗證 `@line/line-bot-mcp-server`
- 查 LINE OA 訊息配額、推送文字/Flex、廣播、Rich Menu
- 修 `create_rich_menu`（Windows Chrome / `file://` / chatBar 長度 / 失敗殘留選單）
- 解釋 `get_follower_ids` 403／「Access to this API is not available for your account」
- 修全域 npm/`npx -y` 因 `make-fetch-happen/lib/cache` 缺檔而崩潰
- 把 LINE Channel Access Token 接到 pi MCP

不要用於：GAS LINE webhook（用 `gas-line-gemini-bot`）、一般 Playwright MCP、Shopee 爬蟲。

## Canonical local layout

| 項目 | 路徑 |
|---|---|
| pi MCP 設定 | `C:/Users/Administrator/.config/mcp/mcp.json`（`mcpServers.line-bot`） |
| 持久安裝（修補後） | `C:/Users/Administrator/.local/mcp/line-bot-mcp-server/` |
| 套件本體 | `.../node_modules/@line/line-bot-mcp-server/`（npm `0.5.0`） |
| 本 skill 修補檔 | `patches/createRichMenu.js`、`patches/getFollowerIds.js` |
| Chrome | `C:/Program Files/Google/Chrome/Application/chrome.exe` |
| 上游 | https://github.com/line/line-bot-mcp-server （preview；README 可能比 npm 新） |

目前可用的 MCP entry 形狀：

```json
{
  "mcpServers": {
    "line-bot": {
      "command": "C:/Program Files/nodejs/node.exe",
      "args": [
        "C:/Users/Administrator/.local/mcp/line-bot-mcp-server/node_modules/@line/line-bot-mcp-server/dist/index.js"
      ],
      "env": {
        "CHANNEL_ACCESS_TOKEN": "<long-lived-token-without-channel-id-prefix>",
        "PUPPETEER_EXECUTABLE_PATH": "C:/Program Files/Google/Chrome/Application/chrome.exe"
      },
      "lifecycle": "lazy"
    }
  }
}
```

不要用 `npx -y @line/line-bot-mcp-server` 當正式入口：本機 Roaming npm 12 曾缺 `make-fetch-happen/lib/cache/policy.js`，且 npx 每次可能覆蓋本機修補。`DESTINATION_USER_ID` 可省略；省略時推送必須帶 `userId`。

## Tools（npm 0.5.0 實際有 12 個）

| 工具 | 作用 | 本機狀態 |
|---|---|---|
| `get_message_quota` | 月配額與已用量 | 可用。此 OA 為 limited 200。 |
| `push_text_message` | 對單一 userId 推文字 | 可用。`message.text` 必填。 |
| `push_flex_message` | 推 Flex（bubble/carousel） | 可用。需 `altText` + `contents`。 |
| `broadcast_text_message` / `broadcast_flex_message` | 對所有好友廣播 | API 通；消耗配額＝好友數。無好友時仍可能計次，勿當測試亂打。 |
| `get_profile` | 暱稱/頭像/狀態/語言 | 可用。需真實 userId。 |
| `get_follower_ids` | 好友 userId 清單 | **帳號層 403**。端點 `GET /v2/bot/followers/ids`。 |
| `get_rich_menu_list` | 列出 Rich Menu | 可用。 |
| `create_rich_menu` | 依 1–6 個 action 產圖、上傳、設預設 | **需套用本 skill 修補**。 |
| `set_rich_menu_default` / `cancel_rich_menu_default` / `delete_rich_menu` | 預設選單與刪除 | 可用。`create` 失敗時修補版會自動刪殘留。 |

README 的 `get_group_summary` **不在 0.5.0**（上游 2026-09-10 才合進 master，尚未發 npm）。

`create_rich_menu` 的 message action schema 是 `{ type: "message", label, text }`，**不是** `message` 欄位。

## Procedure

### 1. 確認 token 與配額

使用者若貼 `2006734807,<base64>`，**只取逗號後面**當 `CHANNEL_ACCESS_TOKEN`。完整字串會 Authentication failed。

```powershell
$tok = (Get-Content -Raw "$env:USERPROFILE\.config\mcp\mcp.json" | ConvertFrom-Json).mcpServers.'line-bot'.env.CHANNEL_ACCESS_TOKEN
Invoke-RestMethod -Headers @{ Authorization = "Bearer $tok" } -Uri https://api.line.me/v2/bot/message/quota
Invoke-RestMethod -Headers @{ Authorization = "Bearer $tok" } -Uri https://api.line.me/v2/bot/message/quota/consumption
```

成功例：`{"type":"limited","value":200}` + `{"totalUsage":0}`。

### 2. 安裝持久套件（不要靠 npx）

```powershell
$dir = "$env:USERPROFILE\.local\mcp\line-bot-mcp-server"
New-Item -ItemType Directory -Force $dir | Out-Null
Set-Location $dir
npm init -y
npm install @line/line-bot-mcp-server@0.5.0 --no-fund --no-audit
```

Node 需 ≥ 22。套件路徑：

```text
C:\Users\Administrator\.local\mcp\line-bot-mcp-server\node_modules\@line\line-bot-mcp-server\dist\index.js
```

### 3. 套用 Windows 修補

把本 skill 的 patches 蓋到套件 `dist/tools/`：

```powershell
$pkg = "$env:USERPROFILE\.local\mcp\line-bot-mcp-server\node_modules\@line\line-bot-mcp-server\dist\tools"
$src = "D:\OB\skills\line-bot-mcp\patches"
Copy-Item "$src\createRichMenu.js" "$pkg\createRichMenu.js" -Force
Copy-Item "$src\getFollowerIds.js" "$pkg\getFollowerIds.js" -Force
```

修補內容（相對上游 0.5.0）：

1. `chatBarText` 截到 14 字（LINE 上限）；`name` 可保留原文。
2. 建立時 `selected: false`，圖上傳成功後再 `setDefaultRichMenu`。
3. 失敗且已有 `richMenuId` 時刪掉殘留選單。
4. `page.goto(pathToFileURL(tempHtmlPath).href)`，修 Windows `file://C:\...`。
5. 預設用本機 Chrome；也可由 `PUPPETEER_EXECUTABLE_PATH` 覆蓋。
6. 拿掉硬編碼 `/tmp` 複製（Windows 沒這個目錄）。
7. `get_follower_ids` 403 時回傳帳號限制說明，不要假裝能列出好友。

**不要**直接改 npx cache（`%LOCALAPPDATA%\npm-cache\_npx\...`）；下次 `npx -y` 會蓋掉。

### 4. 寫入 mcp.json（保留其他 server）

1. 讀並解析現有 JSON。
2. 只改 `mcpServers.line-bot`。
3. 備份到 `%TEMP%\pi-work\line-bot-mcp\`，不要放進 vault。
4. `command` 用 `C:/Program Files/nodejs/node.exe`，`args` 指到步驟 2 的 `dist/index.js`。
5. env 必填 `CHANNEL_ACCESS_TOKEN`、`PUPPETEER_EXECUTABLE_PATH`。
6. 驗證 JSON 後重啟 pi session（MCP schema 不會熱更新）。

不要把 token 寫進 skill、commit 或聊天紀錄以外使用者已提供的當下回合。

### 5. 驗證

stdio 最小握手（在套件目錄外執行亦可）：

```powershell
$entry = "$env:USERPROFILE\.local\mcp\line-bot-mcp-server\node_modules\@line\line-bot-mcp-server\dist\index.js"
# 設好 CHANNEL_ACCESS_TOKEN 與 PUPPETEER_EXECUTABLE_PATH 後：
# initialize → tools/list 應有 12 個工具
# get_message_quota 應回 limited/totalUsage
```

`create_rich_menu` 驗證（會暫時改預設選單，測完必須刪）：

- actions 用 `{ "type":"message","label":"Hi","text":"hello" }`
- 成功後 `cancel_rich_menu_default` + `delete_rich_menu`
- `get_rich_menu_list` 確認沒有 `mcp-fix` / `mcp-self-test` 殘留

`get_follower_ids` 在此 OA 預期 403；修補後訊息應提到 plan/account restriction。

pi 當前 session 若 `mcp({ connect: "line-bot" })` 顯示 not found，是 adapter 啟動快取，重開 pi 即可，不必重裝。

## Rules and Limitations

- 廣播與大量 push 會吃月配額（此帳 200）。測試優先用 `get_message_quota` / `get_rich_menu_list`。
- 不要為了測 `get_follower_ids` 換 token、清 cookie、或宣稱已繞過 LINE 限制。
- 不要改使用者既有生產 Rich Menu（名稱含 cbm-* 等）當測試品；測試選單必須刪掉。
- npm 全域 `AppData/Roaming/npm` 壞掉時，優先修 `make-fetch-happen/lib/cache/*.js`，或改走持久 `node dist/index.js`，不要升級亂蓋其他全域套件。
- 上游更新 npm 後，patches 可能失效；重裝 0.5.0 後必須再 copy patches。若升級到含 `get_group_summary` 的版本，先 diff 再移植修補，不要整檔覆蓋不相容的 `createRichMenu.js`。

## Pitfalls

- Token 帶 channel id 前綴 → Authentication failed。
- `create_rich_menu` 用 `message:` 而非 `text:` → MCP `-32602` Invalid arguments。
- puppeteer 預設 Chrome 148 不在 `~/.cache/puppeteer` → 必須指定系統 Chrome。
- `page.goto('file://' + winPath)` 在 Windows 會壞；要用 `pathToFileURL`。
- chatBar > 14 字 → LINE 400。
- `create` 半成功會留下無圖選單；未修補版不會自動清。
- `/v2/bot/followers`（沒有 `/ids`）是錯路徑；正確是 `/v2/bot/followers/ids`。403 仍可能是帳號方案。
- 改 `mcp.json` 後不重開 pi → 工具清單仍是舊的 2 個 server。
- 此機器 `C:\Users\HCH` 常是 junction；設定以實際檔案與 `D:\.system` 為準，但目前這台 Administrator 的 MCP 檔在 `C:\Users\Administrator\.config\mcp\mcp.json`。

## Verification

1. `python -m json.tool "$env:USERPROFILE\.config\mcp\mcp.json"` 成功。
2. `Test-Path` 套件 `dist/index.js` 與 Chrome exe。
3. `createRichMenu.js` 含 `pathToFileURL` 與 `slice(0, 14)`。
4. LINE API quota HTTP 200。
5. `create_rich_menu` 成功後測試選單已刪、生產選單仍在。
6. 重開 pi 後 `mcp({})` 看得到 `line-bot`。

## Bundled files

- `patches/createRichMenu.js`：0.5.0 Windows 修補版（覆蓋 `dist/tools/createRichMenu.js`）。
- `patches/getFollowerIds.js`：403 時附帳號限制說明。

## Conformance Addendum

## Inputs and Outputs
- **Input:** LINE long-lived Channel Access Token、現有 `mcp.json`、本機 Chrome、npm 0.5.0 套件。
- **Output:** 可啟動的 `line-bot` MCP、配額/選單查詢證據、修補後的 create_rich_menu 成功或明確的 follower API 帳號限制。

## Rules and Limitations
- 相對路徑以本 Skill 資料夾為準。
- 不把 token 寫進 skill 或索引。
- 不重寫其他 MCP server。

## Pitfalls
- 不要把 README 的 13 個工具當成已發布 npm 清單。
- 不要把 403 follower API 當成安裝失敗去重裝。

## Verification
- quota API、tools/list=12、create_rich_menu 成功並清理、mcp.json 仍含 windows-mcp / playwright。
