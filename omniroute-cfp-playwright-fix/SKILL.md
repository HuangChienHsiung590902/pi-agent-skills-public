---
name: "omniroute-cfp-playwright-fix"
description: "修復 OmniRoute CFP（Cloudflare Playground）provider 的 502/Playwright Chromium 瀏覽器缺失錯誤。當 PI-Desktop 或其他透過 OmniRoute 使用 cfp/* 模型（如 cfp/deepseek-ai/deepseek-r1-distill-qwen-32b）時出現「將在 X 秒後重試」「browserType.launch: Executable doesn't exist」「chromium_headless_shell 路徑不存在」等症狀時，使用此 skill。也適用於 OmniRoute 回傳 502 bad_gateway 且錯誤訊息包含 ms-playwright 路徑的任何情境。"
---
# OmniRoute CFP Playwright Chromium Fix

## When to Use

- PI-Desktop（或其他透過 OmniRoute 呼叫 LLM 的客戶端）出現「將在 X 秒後重試 · 第 N/10 次」的自動重試循環
- OmniRoute `/v1/chat/completions` 回傳 HTTP 502 且錯誤訊息包含 `browserType.launch: Executable doesn't exist`
- 錯誤路徑指向 `C:\Users\<user>\AppData\Local\ms-playwright\chromium_headless_shell-<version>\chrome-headless-shell-win64\chrome-headless-shell.exe`
- 使用了 `cfp/*` 前綴的模型（如 `cfp/deepseek-ai/deepseek-r1-distill-qwen-32b`）

## Background

OmniRoute 的 `cfp`（Cloudflare Playground）provider 背後透過 Playwright 操控 headless Chromium 來與 Cloudflare 互動。若系統上缺少對應版本的 Playwright Chromium 瀏覽器執行檔，OmniRoute 會因為無法啟動瀏覽器 session 而回傳 502，PI-Desktop 則進入自動重試循環（預設最多 10 次）。

## Inputs and Outputs

### Inputs

- 確認 OmniRoute 服務 URL（預設 `http://localhost:20128`）與 API key
- 確認發生錯誤的模型 ID（`cfp/*` 前綴）

### Outputs

- Playwright Chromium headless shell 瀏覽器安裝完成
- 若版本號不匹配，建立 junction/symlink 橋接
- OmniRoute `/v1/chat/completions` 回傳 200 成功

## Procedure

### Step 1：確認根因

先直接呼叫 OmniRoute API 確認錯誤：

```bash
# 方法一：用 curl（需 API key）
curl -s http://localhost:20128/v1/chat/completions \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"cfp/deepseek-ai/deepseek-r1-distill-qwen-32b","messages":[{"role":"user","content":"hi"}],"max_tokens":50}'
```

若回傳 502 且 message 包含 `chromium_headless_shell` 路徑不存在，則確認是本問題。

也可從 PI-Desktop 的 CDP debug port 直接讀取頁面內容確認：

```bash
# 先以 debug 模式啟動 PI-Desktop：
# PI-Desktop.exe --enable-logging --v=1 --remote-debugging-port=9222
# 然後讀取頁面文字：
node -e "
const ws = new WebSocket('ws://localhost:9222/devtools/page/<page_id>');
ws.onopen = () => ws.send(JSON.stringify({id:1, method:'Runtime.evaluate', params:{expression:'document.body.innerText', returnByValue:true}}));
ws.onmessage = (e) => { const m = JSON.parse(e.data); if(m.id===1) console.log(m.result.result?.value); ws.close(); process.exit(0); };
setTimeout(()=>process.exit(0), 3000);
"
```

### Step 2：取得 OmniRoute API key

Api key 通常存放在 OmniRoute agent extension 設定檔：

```
C:\Users\<user>\.pi\agent\omniroute-agent-extension\config.json
```

格式為 `{"serverUrl":"http://127.0.0.1:20128","apiKey":"sk-..."}`。

### Step 3：安裝 Playwright 與 Chromium headless shell

系統 npm 若損壞（常見於 Node.js 升級後），改用 corepack pnpm：

```powershell
# 建立暫存目錄並用 pnpm 安裝 Playwright
mkdir C:\Users\Administrator\tmp_playwright
cd C:\Users\Administrator\tmp_playwright
corepack pnpm add playwright

# 安裝 Chromium headless shell 瀏覽器
node -e "
const { execSync } = require('child_process');
execSync('node node_modules/playwright/cli.js install chromium-headless-shell', {
  encoding: 'utf8', stdio: 'inherit', timeout: 180000
});
"
```

### Step 4：處理版本號不匹配

OmniRoute 預期特定版本（如 `chromium_headless_shell-1234`），但 Playwright 當前安裝的可能版本不同（如 `1243`）。檢查 `C:\Users\<user>\AppData\Local\ms-playwright\` 目錄，若已安裝的版本與錯誤訊息中的版本不同，建立 junction 橋接：

```powershell
New-Item -Path 'C:\Users\Administrator\AppData\Local\ms-playwright\chromium_headless_shell-<NEEDED_VER>' `
  -ItemType Junction `
  -Target 'C:\Users\Administrator\AppData\Local\ms-playwright\chromium_headless_shell-<INSTALLED_VER>' `
  -Force
```

確認 junction 有效：

```powershell
Test-Path 'C:\Users\Administrator\AppData\Local\ms-playwright\chromium_headless_shell-<NEEDED_VER>\chrome-headless-shell-win64\chrome-headless-shell.exe'
```

### Step 5：驗證修復

再次呼叫 OmniRoute chat completion API（同 Step 1），確認回傳 HTTP 200 且有正常回應內容。

然後在 PI-Desktop 開新對話（點擊工具列「新建任務」按鈕），輸入簡單測試問題並確認能正常獲得回覆。不要使用先前失敗的對話（已被重試耗盡）。

## Rules and Limitations

- 不得刪除 `ms-playwright` 下的任何現有瀏覽器目錄；只建立 junction 橋接
- Junction 目標必須是已安裝的正確版本目錄，不可指向不存在路徑
- 若 npm 全域損壞，優先使用 pnpm（`corepack pnpm`）而非嘗試修復 npm
- 此問題只發生在 `cfp/*` provider；若使用其他 provider（qct/、qwen-cloud-token-plan/ 等）則無關
- 不要動態修改 OmniRoute 設定或更換 API key 來嘗試規避問題

## Pitfalls

- **不要直接按 npm 官方文件用 `npx playwright install`**：若系統 npm 損壞（`Cannot find module './cache/policy.js'`），會白白浪費時間。改用 `corepack pnpm add playwright` 再手動執行 cli.js。
- **不要忽略版本號不匹配**：即使 Playwright 已安裝成功，OmniRoute 可能綁定特定舊版本號（如 1234），必須建立 junction。
- **不要急著重啟 OmniRoute 或 PI-Desktop**：修復瀏覽器路徑後 OmniRoute 會在下一次請求時自動使用，不需要重啟服務。
- **不要用 `mklink` 在 bash 裡執行**：Windows bash 與 cmd 的互動有路徑轉譯問題，一律使用 PowerShell `New-Item -ItemType Junction`。

## Verification

1. `Test-Path` 確認目標路徑的 `chrome-headless-shell.exe` 存在
2. 直接 curl OmniRoute `/v1/chat/completions` 回傳 200
3. PI-Desktop 新對話可正常獲得模型回覆，無重試循環
4. `C:\Users\<user>\AppData\Local\ms-playwright\` 下存在 junction 連結且指向正確版本