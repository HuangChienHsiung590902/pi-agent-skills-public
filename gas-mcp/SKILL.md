---
name: gas-mcp
description: 管理本機的 Google Apps Script MCP server（whichguy/mcp_gas，安裝於 C:\Users\HCH\mcp-servers\gas-mcp）——重新 build、處理 OAuth 授權、排解 port 3000 卡住/session 斷線問題、以及上游 repo 更新後要重新套用的已知 Windows/OAuth bug 修正。當使用者要求「gas mcp」、「Google Apps Script MCP」、「gas 授權」、「gas 連不上」、「重裝 gas-mcp」、或 gas 相關工具（mcp__gas__*）出錯時使用。
---

# gas-mcp（Google Apps Script MCP Server）管理

## 快速參考

| 項目 | 值 |
|------|-----|
| 安裝路徑 | `C:\Users\HCH\mcp-servers\gas-mcp`（clone 自 `https://github.com/whichguy/gas_mcp`，upstream 實際專案名是 `mcp_gas`） |
| 註冊方式 | `claude mcp add -s user gas -e NODE_ENV=production -- node C:/Users/HCH/mcp-servers/gas-mcp/dist/src/index.js --config C:/Users/HCH/mcp-servers/gas-mcp/gas-config.json` |
| Build | `cd ~/mcp-servers/gas-mcp && npm run build` |
| OAuth callback port | 固定 `3000`（寫死在 `src/auth/oauthClient.ts`，不是隨機 port） |
| 已授權帳號 | hch590902@gmail.com（token 存於 `~/.auth/mcp-gas/tokens/`） |
| GCP 專案 | `gas-mcp-503101`（自建，Testing 發布狀態，見下方 OAuth 用戶端） |
| GCP 已啟用 API | Apps Script、Drive、Sheets、Docs（要用 Calendar/Gmail/Slides 等其他服務要自己去 API Library 手動啟用） |
| OAuth client_id | `885565514803-211m9t0eja7er8id9vpc7iafn7o5jcnu.apps.googleusercontent.com`（電腦版應用程式類型） |
| 相關記憶 | `project_gas_mcp_install`（完整踩坑記錄） |

## 現況（2026-07-21 驗證完成）

已完整安裝、修好全部已知 bug、OAuth 授權成功，`mcp__gas__*` 共 36 個工具可用（`cat`/`write`/`exec`/`deploy`/`git_feature`/`project_create`...）。**已在既有專案上實測 `write` → `require()` → 拿到正確結果，讀寫執行全部正常，可以放心拿來開發**（唯一例外見下方第 8 點：`project_create` 全新專案目前壞掉，開發請用既有專案）。**遇到問題前不要重新 clone 或砍掉重裝**——先看下面「已知 bug 與修正」是不是同一批問題復發。

## 已知 bug 與修正（若重新 `git pull` 更新 upstream，這些會全部復發，要重新套用）

upstream 專案本身沒針對 Windows 測試過，且作者自己的公開 OAuth client 是給他自己的測試帳號用的，外人用不了。踩過的坑，按修正順序列出：

1. **Windows 路徑解析壞掉導致完全啟動不了（無任何錯誤訊息）**
   `src/config/shimTemplate.ts`、`src/utils/templateLoader.ts`、`src/utils/codeGeneration.ts` 用 `new URL(import.meta.url).pathname` 取檔案路徑，Windows 上會多一個磁碟機代號前的斜線（`/C:/Users/...`），導致路徑被錯誤拼接成 `C:\C:\...`。
   → 改用 `fileURLToPath(import.meta.url)`（`node:url`），`dist`/`src` 邊界判斷改用正則 `/[\\/]dist[\\/]/`。

2. **`main()` 從未被呼叫**
   `src/index.ts` 最後判斷「是否直接執行本檔」用 `import.meta.url === \`file://${process.argv[1]}\``，Windows 上因為正反斜線不一致永遠不成立。
   → 改成 `process.argv[1] && fileURLToPath(import.meta.url) === path.resolve(process.argv[1])`。

3. **OAuth 逾時後 port 3000 卡死，下次授權必定 `EADDRINUSE`**
   `src/auth/oauthClient.ts` 的 callback server 固定綁 port 3000；`src/tools/auth.ts` 的 5 分鐘逾時處理只 reject promise，沒關掉這個 server。
   → 讓 `GASAuthClient.cleanupServer()` 改成 public，並在 auth.ts 的逾時 callback 裡呼叫它。
   → **若又遇到 `EADDRINUSE: address already in use 127.0.0.1:3000`**：`netstat -ano | grep ":3000" | grep LISTENING` 找 PID，確認是 `node .../gas-mcp/dist/src/index.js` 後 `Stop-Process -Force` 砍掉，讓它自動重啟。

4. **client_id 寫死在原始碼，改設定檔完全沒用**
   `src/tools/authConfig.ts` 的 `loadOAuthConfigFromJson()`（函式名誤導）其實是硬編碼回傳，完全不讀 `gas-config.json`/`oauth-config.json`。原本用的是作者自己 GCP 專案的公開 client，屬於「測試中」狀態、白名單只有他自己的帳號，外部帳號登入會被 Google 擋下 `403 access_denied`。
   → 已在 Google Cloud Console 建立專屬 GCP 專案 `gas-mcp-503101`，啟用 Apps Script API，OAuth 同意畫面設為外部/測試中，測試使用者加了 `hch590902@gmail.com` 和 `hch.new@gmail.com`，建立「電腦版應用程式」類型的 OAuth 用戶端。新 client_id 已寫進 `authConfig.ts` 和 `src/auth/sessionManager.ts`（token refresh 用的另一個硬編碼位置）。
   → **要加第三個測試帳號**：登入 https://console.cloud.google.com/auth/audience?project=gas-mcp-503101 → 測試使用者 → 新增使用者。加完務必**整頁重新整理**確認真的存進去了（UI 曾經點了儲存但沒真的持久化，見下一條）。

5. **「Desktop app」類型仍然會發 client_secret，沒帶上會 token exchange 失敗**
   Google 對 Desktop app OAuth client 還是會產生一組 `client_secret`（雖然文件說不是機密性用途），沒有在 token exchange 帶上會回 `Token Exchange Failed: invalid_request`。
   → 已把 client_secret 加進 `authConfig.ts` 回傳的設定裡。

6. **非阻塞模式 (`waitForCompletion:false`) 的 session 同步只等 10 秒**
   使用者真的點完 Google 帳號選擇+同意畫面通常要超過 10 秒，逾時一到 resolver 就被刪掉，之後瀏覽器顯示「Authentication Successful」但 server 端 session 從沒寫進去（`~/.auth/mcp-gas/tokens/` 是空的）。
   → 已把這個逾時延長到 300000ms（5 分鐘，跟 blocking 模式一致）。
   → **權宜作法**：呼叫 `auth(mode:"start")` 時**直接用預設值（不要傳 `waitForCompletion:false`）**，讓它走 blocking 模式，工具呼叫超過 120 秒會自動轉背景執行，等通知就好，比較不會踩到這個坑。

7. **GCP 專案一開始只啟用了 Apps Script API，缺 Drive/Sheets/Docs API**
   `project_create`/`ls` 等工具會噴 `403 Google Drive API has not been used in project ... or it is disabled`（GAS 專案本身存在 Drive 裡）。
   → 已到 https://console.cloud.google.com/apis/library/drive.googleapis.com?project=gas-mcp-503101 （sheets/docs 同理換網址）點「啟用」。之後如果要用 Calendar/Gmail/Slides 等其他 GAS 服務，一樣先去 API Library 手動啟用對應 API。

8. **`project_create` 建立的全新專案，CommonJS 基礎設施裝不起來**
   新專案 `exec`/`require()` 一律報 `require is not defined`，`project_create` 回應裡的 `infraErrors` 當下就有警告。試過 `reorder`（把 require.gs 排到 position 0）、`deploy_config({operation:"reset"})` 都沒用，**根因還沒找到**。
   → **權宜作法：先用 `project_list` 看有沒有既有專案可以借來開發，避免用 `project_create` 開全新專案**，直到這個 bug 被抓出來。

## 重新 build 流程

```bash
cd ~/mcp-servers/gas-mcp
npm run build
```

Build 完只是更新 `dist/` 檔案，**正在跑的 node process 記憶體裡還是舊程式碼**，一定要找到並砍掉舊 process 才會生效：

```bash
netstat -ano | grep ":3000" | grep LISTENING   # 若 OAuth 流程剛好卡著，port 3000 會有殘留 PID
```

或直接找 process：

```powershell
Get-CimInstance Win32_Process -Filter "CommandLine LIKE '%gas-mcp/dist/src/index.js%'" | Select-Object ProcessId,CommandLine
Stop-Process -Id <PID> -Force
```

## ⚠️ 重啟 gas server process 後，本次對話會跟工具斷線

這是本機 Claude Code harness 的已知行為，不是 gas-mcp 本身的問題：只要 `Stop-Process` 砍掉 gas server 的 node process，這個對話 session 裡的 `mcp__gas__*` 工具會立刻變成「MCP server disconnected」，即使 `claude mcp list` 顯示它已經重新連線健康。

**目前唯一可靠的解法：使用者要完整重新啟動 Claude Code app（不是打字 "continue" 就有用）。** 重啟後系統會自動跳出 system-reminder 說 36 個 `mcp__gas__*` 工具重新可用，這時再用 `ToolSearch` 找 `mcp__gas__*` 就能抓到。

**因此改 gas-mcp 原始碼、重新 build、需要重啟 process 的操作，一次對話裡盡量一次做完、集中處理，避免使用者要反覆重啟 App。**

## OAuth 授權流程

```
mcp__gas__auth({ mode: "status" })   # 先查有沒有登入
mcp__gas__auth({ mode: "start" })    # 沒登入就開始（用預設 waitForCompletion，見上面第 6 點）
```

`mode:"start"` 會用系統預設瀏覽器開登入視窗（不是 Claude 接管的 CDP Chrome，playwright 看不到），使用者要自己在跳出來的視窗完成 Google 登入。完成後這個工具呼叫就會回傳 `authenticated: true`。

驗證是否真的授權成功：

```bash
ls -la ~/.auth/mcp-gas/tokens/    # 應該有 <email>.json
```

## 換帳號 / 登出

```
mcp__gas__auth({ mode: "logout" })
```

## 已驗證：可以正常開發程式碼（2026-07-21）

在既有專案（例如 scriptId `1mB-Ly4BSRvk7QqSa3qsThDUdN5667I5SZYjejpOS3ou4YEt_utKNC8k1`）上完整測過 `write` → `require()` 呼叫 → 拿到正確結果，讀寫執行全部正常。開發前務必看過上面第 7、8 點的兩個額外坑。

---

## Conformance Addendum

## When to Use
管理本機的 Google Apps Script MCP server（whichguy/mcp_gas，安裝於 C:\Users\HCH\mcp-servers\gas-mcp）——重新 build、處理 OAuth 授權、排解 port 3000 卡住/session 斷線問題、以及上游 repo 更新後要重新套用的已知 Windows/OAuth bug 修正。當使用者要求「gas mcp」、「Google Apps Script MCP」、「gas 授權」、「gas 連不上」、「重裝 gas-mcp」、或 gas 相關工具（mcp__gas__*）出錯時使用。

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
