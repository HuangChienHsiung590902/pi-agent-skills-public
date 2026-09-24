---
name: tauri-wiki-app
description: 管理「LLM Wiki」這款獨立安裝的 Tauri 桌面 wiki app(REST API port 19828)——查詢/新增/編輯 wiki 頁面、搜尋內容、取得知識圖譜、透過 `/chat` 的隱藏 `images` 欄位傳圖做 OCR/影像辨識、幫 bundled mcp-server 補圖片支援；也處理 OCR 報 `At most 0 image(s)`、401、看不到圖片。當使用者提到「查詢 wiki」「管理 wiki 頁面」「搜尋 wiki」「新增 wiki 條目」「llm wiki 專案」「llm wiki 傳圖」「llm wiki OCR」「llm wiki 看不到圖片」時使用。⚠️ 跟 OMC 內建的 `wiki` skill(讀寫 `.omc/wiki/*.md`)是完全不同的兩套系統，不要混淆。
version: 1.2.0
---

# tauri-wiki-app Skill（原名 llm-wiki）

「LLM Wiki」是一款本地知識庫系統，基於 Tauri 建構，透過 REST API 提供 wiki 管理功能。**跟 OMC 內建的 `wiki` skill 是兩個完全獨立的系統**（那個是純 markdown 檔案存在 `.omc/wiki/`，這個是有自己資料庫/專案模型的獨立桌面 app），名字容易搞混但底層完全無關。

## 基本資訊

- **API 端點**：`http://127.0.0.1:19828/api/v1`
- **健康檢查**：`/health`（無需授權）
- **授權方式**：實測（2026-07-29，用官方 bundled mcp-server 的 `api-client.js` 為準）是
  Header `Authorization: Bearer <token>`，不是舊版寫的 `X-LLM-Wiki-Token`（可能是
  API 演進後改了，或原本就記錯；以 `Bearer` 為準）。Token 存在
  `~/.config/mcp/mcp.json` 的 `mcpServers.llm-wiki.env.LLM_WIKI_API_TOKEN`
  （跟 pi/opencode 共用同一組，見 [[connect-llm-wiki]]）。

### apiConfig 設定與 token 實測細節（2026-07-30）

app 自己的 auth 設定存在 `%APPDATA%\com.llmwiki.app\app-state.json` 的
`apiConfig` 區塊：

```json
"apiConfig": {
  "allowLanAccess": false,
  "allowUnauthenticated": true,
  "enabled": true,
  "mcpEnabled": true,
  "token": "ceadcfa3ce426248f702777875aa24ef2830bcf298bd8a6c"
}
```

實測發現的行為（跟原本猜測不完全一樣）：

- 這個檔案是**即時生效**的——改完 `token`/`allowUnauthenticated`/`mcpEnabled`
  不用重啟 app，下一個 request 就會用新設定（至少對 app 自己的 HTTP server
  是如此）。
- **`allowUnauthenticated: true` 只對唯讀端點生效**（實測 `/projects` 不帶任何
  Authorization header 就直接 200），**`/chat` 端點即使 `allowUnauthenticated`
  是 true，沒帶有效 `Authorization: Bearer <token>` 一樣回 401**——研判是
  `/chat` 會觸發 LLM 呼叫（有成本／會寫入 session 檔案），刻意排除在
  「免驗證」的範圍外，需要 `token` 欄位有實際值才行。
- 目前 app 的 Settings UI 裡沒有找到「產生 token」的按鈕（畫面上只看到
  Base URL，token 欄位是空的）——上面這組 token 是直接手動寫進
  `app-state.json` 生出來的（`secrets.token_hex(24)`），不是從 UI 複製的。
  如果 app 之後補了產生 token 的 UI，用那邊產生的可能更保險，但目前這組
  手動塞的已驗證可正常用於 `/projects` 和 `/chat`。
- 這組 token 目前只在 app-state.json 這一份設定裡，跟
  `~/.config/mcp/mcp.json` 的 `LLM_WIKI_API_TOKEN`（給官方 bundled
  mcp-server 用）是**兩個獨立的設定位置**，沒有自動同步，換一邊要記得
  另一邊要不要跟著改。
- **`D:/MCP/llm-wiki-mcp/index.js` 是另一套自製的 MCP server**（跟上面
  「官方 bundled mcp-server」是不同程式碼），用的是 `X-LLM-Wiki-Token`
  header（不是 `Bearer`），token 在 Node process **啟動時讀取一次**
  `app-state.json` 的 `apiConfig.token`，之後就算檔案內容改了也不會重讀
  ——要讓這個 MCP server 吃到新 token，必須重啟該行程（例如重啟 Claude
  Code）。這個是 Claude Code 這個 session 裡實際掛的 llm-wiki MCP 工具
  （`mcp__llm-wiki-mcp__*`）。

## 功能模組

| Command | 用途 |
|---------|------|
| `/llm-wiki-projects` | 列出所有專案，取得當前專案 |
| `/llm-wiki-files` | 瀏覽、新增、編輯、刪除 wiki 頁面 |
| `/llm-wiki-search` | 搜尋 wiki 內容 |

## 專案結構

```
skills/llm-wiki/
├── SKILL.md              # 本檔案
├── commands/
│   ├── llm-wiki-projects.md
│   ├── llm-wiki-files.md
│   └── llm-wiki-search.md
├── scripts/
│   └── scripts/llm-wiki-api.ps1  # API 呼叫 helper
└── output/               # 查詢結果輸出
```

## 常用 API 端點

| 方法 | 端點 | 說明 |
|------|------|------|
| GET | `/projects` | 列出所有專案 |
| GET | `/projects/{id}/files?root=wiki` | 列出 wiki 頁面 |
| GET | `/projects/{id}/files/content?path=wiki/xxx.md` | 取得檔案內容 |
| POST | `/projects/{id}/search` | 搜尋 |
| POST | `/projects/{id}/chat` | 問後端 Agent 問題（見下方「傳圖片/OCR」） |
| GET | `/projects/{id}/graph` | 取得知識圖譜 |

## 注意事項

- 專案 ID 可用 `current` 代表當前專案
- 唯讀端點：`/health`
- 需要授權的端點：除 `/health` 外全部需要

## 傳圖片給 chat（`/chat` 的 `images` 欄位，官方沒寫在任何文件裡）

`POST /projects/{id}/chat` 除了 `message`/`sessionId`/`mode`/`tools` 這些
bundled mcp-server 的 `api-client.js` 原本就有實作的欄位外，**還接受一個完全沒
被那份 client 程式碼串接、也沒寫在 GitHub README 的 `images` 欄位**：

```json
{
  "message": "請OCR這張圖片，把裡面的文字內容列出來",
  "tools": { "wiki": false, "web": false, "anytxt": false },
  "images": [
    { "dataBase64": "<純 base64，不要 data: 前綴>", "mediaType": "image/png" }
  ]
}
```

**怎麼挖出來的**：後端是 Rust（serde），送錯欄位名會回具體的
`missing field` 錯誤，照著錯誤訊息一步步試出正確欄位名——一開始猜
`images: [{data, name}]` 被打回「missing field \`mediaType\`」，補上後又被
打回「missing field \`dataBase64\`」（不是 `data`）。這招（送一個大致合理
但故意留空/欄位名用猜的 payload，靠 serde 的驗證錯誤逼出正確 schema）在
碰到任何 Rust/Axum/Actix 後端、又沒有 OpenAPI 文件時都好用。

`mediaType` 目前只確認 `image/png` 可用，其他常見圖片格式理論上也支援
（看後端用什麼 image crate 解碼，沒有一一實測）。

回應會經過整個 `/chat` 的「Agent 迴圈」（工具呼叫、wiki/web/anytxt 檢索等），
**不是**單純的一次性 vision completion，所以：
- 會受 `tools.wiki/web/anytxt` 開關影響（跟平常呼叫 `/chat` 一樣，OCR 用途通常
  三個都關掉，避免模型分心去搜尋知識庫）
- 這個 Agent 迴圈本身有自己的 JSON tool-call 協定，跟 reasoning 模型（例如
  Qwythos，見 [[vllm-qwythos-deploy]]）搭配偶爾會不穩定——同一組完全正確的
  request，有時候模型會被自己的工具呼叫協定卡住、回「偵測不到圖片」或
  「Invalid/truncated Agent tool JSON」，換一次呼叫（或簡化 prompt 不要提
  「OCR」這種容易觸發它去找工具的字眼）通常就正常了。這是後端 Agent 邏輯的
  既有不穩定性，不是 `images` 欄位本身的問題——想確認 payload 有沒有真的送對，
  可以先繞過 Agent 迴圈直接測純 API（見下方驗證方式），排除是不是傳輸層出錯。

**⚠️ 前置條件**：App 的 Settings → Image Captioning →
「Enable captioning at ingest」這個開關**必須關掉**，chat 的圖片輸入才會生效
（這個開關文件上寫的是「只影響 PDF/DOCX/PPTX 匯入」，但實測開著會連 chat
傳圖也一起失效，原因不明，可能兩條 pipeline 共用了某段程式碼）。細節見 memory
`reference_llmwiki_image_captioning_toggle`。

**✅ 2026-07-30 端到端驗證通過**：`multimodalConfig.enabled` 設回 `false`
（= 關掉 ingest captioning）之後，直接打 `/chat` 帶一張寫著
"HELLO WORLD 12345" 的測試圖、`tools` 全部 `false`、帶正確 `Authorization:
Bearer <token>`，回應乾淨：`message.content` 正確逐字辨識出文字，
`toolEvents` 只有一筆 `llm.generate`（沒有再誤觸發 wiki/web/anytxt 搜尋，
也沒再出現 `Invalid/truncated Agent tool JSON`）。所以目前已知的完整修法是
兩件事都要做：① 關掉 Image Captioning 的 ingest 開關 ② 該次 `/chat` 呼叫的
`tools.wiki/web/anytxt` 全部關掉。

**⚠️ 2026-08-02 再踩坑：`At most 0 image(s) may be provided`**：這個錯誤
不是 LLM Wiki 的 `images` payload schema 錯，而是上游 OpenAI-compatible LLM
服務宣告自己**最多接受 0 張圖片**。當時直接打
`http://10.145.119.19:8182/v1/chat/completions` 夾 `image_url` 也回同一個
400，最後查到遠端 container `vllm-qwythos` 是用 `--language-model-only`
啟動，移除此參數並重啟 vLLM 後，直接 vLLM OCR 截圖已恢復 200。遇到這個錯
先照順序查：
1. `app-state.json` 的 `multimodalConfig.enabled` 必須是 `false`。
2. `/chat` 要帶有效 `Authorization: Bearer <token>`；`/chat` 不吃免驗證，沒 token
   會 401。若 `apiConfig.token` 空，先產生/補回 token，並開 `apiConfig.enabled`。
3. 繞過 LLM Wiki 直接打 vLLM `/v1/chat/completions` 測 image；若同樣 400，去
   10.145.119.19 檢查 vLLM 啟動參數，不能有 `--language-model-only`。
4. 若 vLLM 直接測圖已 200，但 LLM Wiki 還說「沒圖片」，那才是 LLM Wiki Agent
   loop/JSON 協定不穩；重試或用更明確的 compact JSON final prompt。

## 幫 MCP 工具（`llm_wiki_chat`）補上圖片支援（已 patch，2026-07-29）

官方 bundled 的 mcp-server（`C:\Users\HCH\AppData\Local\LLM Wiki\mcp-server\dist\src\`）
原本沒有把上面這個 `images` 欄位串進 `llm_wiki_chat` 工具，導致透過 MCP
（pi / opencode / Claude Code 的 `mcp__llm-wiki__llm_wiki_chat`）傳訊息時完全
無法夾帶圖片。已手動 patch 這兩個檔案補上：

- **`api-client.js`**：`chat()` 的 request body 加一行 `images: options.images`
  透傳。
- **`index.js`**：
  - `llm_wiki_chat` 工具的 `inputSchema` 新增 `image_paths`（字串陣列，
    **本機絕對路徑**，不是 base64——避免呼叫端要把巨大的 base64 字串塞進
    MCP tool-call 參數，白白吃掉 context）。
  - 新增 `loadImagesFromPaths(paths)` helper：對每個路徑用 `readFileSync`
    讀檔、`toString("base64")` 轉編碼，用副檔名（`.png/.jpg/.jpeg/.webp/.gif/.bmp`）
    對照表推 `mediaType`，組成 `{dataBase64, mediaType}`。
  - 在 `case "llm_wiki_chat"` 的 handler 裡加
    `images: loadImagesFromPaths(stringArrayArg(args.image_paths))`。

**重新套用這個 patch**（App 更新/重灌會蓋掉這兩個檔案，需要重補）：
對照上面的邏輯手動改，或找回這次對話的 diff 重新套用。改完務必：
1. `node --check` 兩個檔案語法
2. 直接 import `LlmWikiApiClient` 呼叫 `.chat()` 帶 `images` 測一次（繞過
   MCP 協定，先確認底層 API 呼叫路徑沒問題）
3. 用 `@modelcontextprotocol/sdk` 的 `Client` + `StdioClientTransport` 真的
   spawn `index.js` 跑一次完整 `tools/list`（確認 schema 有 `image_paths`）
   + `tools/call`（確認真的回傳正確 OCR 內容）——**SDK client 的
   `callTool` 預設 60 秒逾時，Qwythos 這種 reasoning 模型常常不夠，測試時要
   帶 `{ timeout: 240000 }` 這種 options，不然會誤判成失敗**
   （`client.callTool(req, undefined, { timeout: 240000 })`）
4. 測試腳本要放在 `mcp-server/` 目錄底下執行（或用其他方式讓 Node 解析得到
   `node_modules`）——ESM 的模組解析是看**檔案位置**不是看 `cwd`，`cd` 進去再
   跑外面的檔案解析不到 `@modelcontextprotocol/sdk`。

**⚠️ patch 完不會馬上生效**：Node 不會熱重載，任何已經連著這個 MCP server
的行程（Claude Code / pi / opencode）都是用 patch 前的舊程式碼跑的子行程，
要重啟該行程（讓它重新 spawn `node index.js`）才能看到新的 `image_paths`
參數。

---

## Conformance Addendum

## When to Use
管理「LLM Wiki」這款獨立安裝的 Tauri 桌面 wiki app(REST API port 19828)——查詢/新增/編輯 wiki 頁面、搜尋內容、取得知識圖譜、透過 `/chat` 的隱藏 `images` 欄位傳圖做 OCR/影像辨識、幫 bundled mcp-server 補圖片支援；也處理 OCR 報 `At most 0 image(s)`、401、看不到圖片。當使用者提到「查詢 wiki」「管理 wiki 頁面」「搜尋 wiki」「新增 wiki 條目」「llm wiki 專案」「llm wiki 傳圖」「llm wiki OCR」「llm wiki 看不到圖片」時使用。⚠️ 跟 OMC 內建的 `wiki` skill(讀寫 `.omc/wiki/*.md`)是完全不同的兩套系統，不要混淆。

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

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
