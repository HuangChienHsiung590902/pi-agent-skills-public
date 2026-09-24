---
name: gas-line-gemini-bot
description: 在 Google Apps Script（透過 gas-mcp）上開發/除錯「LINE Bot × Gemini API」這類獨立 Webhook 專案。涵蓋 exec() 被專案自己的 doPost 劫持時的繞過技巧、如何用 deploy_config 建立真正可公開觸及的 Web App 部署（而非 deploy() 的函式庫模式）、LINE Webhook 收不到訊息的排查清單、金鑰安全處理原則。當使用者要開發/除錯 GAS 上的 LINE bot、Webhook 型 Apps Script 專案、或提到「LINE bot」「gas webhook」「exec 沒反應」「LINE 沒回應」時使用。
---

# GAS LINE Bot × Gemini API 開發與除錯

依賴 [[gas-mcp]] 技能先確保 gas-mcp 本身能用（授權、build 都正常）才繼續下面的內容。

## 這個專案的現況

| 項目 | 值 |
|------|-----|
| 專案名稱 | `line-gemini-bot-test` |
| scriptId | `1v565rQDjp2HEsn9oGfEafe1ixygjdtu4iZjrNvfRJ_1Kl5as3DwRACAF` |
| 主檔案 | `LineGeminiBot.gs`（純 `doGet`/`doPost` 全域函式，**不是** CommonJS module） |
| Gemini 模型 | `gemini-2.5-flash` |
| 正式 Webhook 網址（prod） | `https://script.google.com/macros/s/AKfycbxLeXXM-NHa0AsLphDmsrqh5QRNlNprdSbBWJ3qq7UjfTKRCIX8vbLUt4CN2qZ41di9/exec` |
| Script Properties | `GEMINI_API_KEY`、`LINE_CHANNEL_ACCESS_TOKEN`（已設定，存在 Script Properties，不在原始碼裡） |

## 坑 1：`exec()` 會被專案自己的 `doPost`/`doGet` 劫持

這種「純標準 Apps Script 網頁應用」專案（沒有跑過 `project_create` 裝好 CommonJS 基礎設施、`cat` 時 `commonJsInfo.moduleUnwrapped` 顯示沒有 wrapper）如果自己定義了全域 `doPost(e)`，`mcp__gas__exec()` 送出的探測請求會被這個 `doPost` 直接攔截、吞掉，回傳的永遠是這支函式自己的回應（例如這裡固定回 `{"status":"ok"}`），**跟你送的 `js_statement` 完全無關**。而且被攔截的那個部署似乎有自己的版本快取，就算改完檔案立刻呼叫 `exec()`，同一支自訂函式在那個 exec 專用部署裡可能還是抓不到（`typeof myFunc` 回傳 `"undefined"`）——這是 gas-mcp 工具本身的已知怪癖，不代表你的程式碼有問題。

**繞過技巧**：想用 `exec()` 檢查/操作這類專案（例如確認 Script Properties 有沒有設對）時，先把 `doPost` 暫時改名，用完再改回來：

```
edit({scriptId, path:"LineGeminiBot.gs", edits:[{oldText:"function doPost(e) {", newText:"function doPost_TEMP_DISABLED(e) {"}]})
exec({scriptId, js_statement:"JSON.stringify(PropertiesService.getScriptProperties().getKeys())"})
edit({scriptId, path:"LineGeminiBot.gs", edits:[{oldText:"function doPost_TEMP_DISABLED(e) {", newText:"function doPost(e) {"}]})
```

改完務必用 `cat({scriptId, path, preferLocal:false})` 核對 hash 跟原始內容一致，確認沒有留下改名的痕跡。

**注意**：`PropertiesService` 讀寫是專案層級的執行期狀態，跟程式碼版本無關，用這招查/設 Script Properties 很可靠；但拿來測試專案自訂函式（像 `callGemini()`）本身的邏輯就不可靠（常常回 `undefined`），那種情況不要花時間跟 `exec()` 死磕，直接看程式碼 review 或改用真正的公開網址發送請求測試。

## 坑 2：`deploy()` 是函式庫模式，不適合單純的 Webhook 專案

`mcp__gas__deploy({to:"staging"/"prod"})` 設計給「函式庫 + 消費端試算表」那種模式（第一次 promote 會自動建立 `-source` 函式庫專案跟一個消費端 Google Sheet），對一支單純要暴露 `/exec` 網址給 LINE 打的 Webhook bot 來說是錯的工具，會建立一堆不需要的東西。

**正確做法**：用 `deploy_config({operation:"status"})` 檢查，如果顯示 `Missing deployments: dev, staging, prod`，直接：

```
deploy_config({operation:"reset", scriptId})
```

這會在**這支腳本自己身上**建立三個標準 Web App 部署（dev/staging/prod），全部指向 HEAD（最新存檔的程式碼），回傳的 `url` 欄位就是可以直接貼給 LINE 當 Webhook 的 `/exec` 網址。`appsscript.json` 裡要先設好：

```json
"webapp": { "executeAs": "USER_DEPLOYING", "access": "ANYONE" }
```

否則外部（LINE 伺服器）打不進來。

## 坑 3：金鑰絕對不要寫死在原始碼裡

只透過 Script Properties 存取金鑰（`PropertiesService.getScriptProperties().getProperty('GEMINI_API_KEY')`），**不要**寫一個 `setupProperties_ONETIME()` 之類的函式把明文金鑰塞進 `.gs` 檔案——這樣任何看得到專案原始碼的人都拿得到金鑰，commit 進 git 後還會永久留在版本歷史裡。要設定金鑰就用坑 1 的繞過技巧透過 `exec()` 呼叫 `setProperty()` 直接寫入 Script Properties，寫完後**不要**把值留在任何 `.gs` 檔案裡。

## 坑 4：LINE 傳訊息完全沒反應時的排查清單

先確認問題出在哪一端：用 `mcp__gas__executions({operation:"list", scriptId, minutes:5})` 看最近有沒有 `doPost` 被觸發過。**如果完全沒有 `doPost` 執行紀錄，代表 LINE 根本沒把請求送到這支腳本**，問題在 LINE 那一側設定，不是程式碼：

1. LINE Developers Console → 你的 Channel → Messaging API → Webhook URL 貼上 `/exec` 網址後有沒有按「Verify」且成功
2. **「Use webhook」開關**有沒有打開
3. 最常忽略：[LINE 官方帳號管理後台](https://manager.line.biz) → 設定 → 回應設定 → **回應模式必須是「機器人」，不能是「聊天」**——設成「聊天」的話訊息會被導去人工聊天視窗，永遠不會觸發 webhook
4. 可以順便把「自動回應訊息」「加入好友的歡迎訊息」關掉，避免跟 bot 的回覆混在一起

如果 `doPost` 有執行紀錄但沒收到 LINE 回覆，才是程式碼/API 金鑰的問題，改用 `mcp__gas__cloud_logs` 或 `executions({operation:"get", processId})` 查詳細錯誤。

## GAS 平台限制（非 bug）

`doGet`/`doPost` 拿不到 HTTP headers，所以沒辦法驗證 LINE 的 `X-Line-Signature`——這是 Apps Script 平台本身的限制，不是這支程式碼的疏漏，代表理論上任何人都能偽造請求打這支 Webhook API，設計上要接受這個限制或改用其他平台（Cloud Functions 等）才能做簽章驗證。

---

## Conformance Addendum

## When to Use
在 Google Apps Script（透過 gas-mcp）上開發/除錯「LINE Bot × Gemini API」這類獨立 Webhook 專案。涵蓋 exec() 被專案自己的 doPost 劫持時的繞過技巧、如何用 deploy_config 建立真正可公開觸及的 Web App 部署（而非 deploy() 的函式庫模式）、LINE Webhook 收不到訊息的排查清單、金鑰安全處理原則。當使用者要開發/除錯 GAS 上的 LINE bot、Webhook 型 Apps Script 專案、或提到「LINE bot」「gas webhook」「exec 沒反應」「LINE 沒回應」時使用。

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
