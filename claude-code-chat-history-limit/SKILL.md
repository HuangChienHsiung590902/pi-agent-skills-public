---
name: claude-code-chat-history-limit
description: 解讀並處理 Claude Code / pi harness 出現「Error 413 chat_history_too_large / payload_too_large」（訊息內容 "Chat history exceeds the 800-message limit; compact the conversation and retry."）錯誤。當使用者遇到這個錯誤訊息、或抱怨「對話卡住/送不出去/一直失敗」且懷疑是對話太長，或要求「處理 800 message limit」「chat history too large」時使用。
---

# Claude Code：對話歷史超過 800 則訊息上限（413 payload_too_large）

## 錯誤長相

```
Error: 413: {"message":"Chat history exceeds the 800-message limit; compact the conversation and
 retry.","type":"payload_too_large","code":"chat_history_too_large","reason":"message_limit"}
```

## 這是什麼

**不是 bug、不是網路問題、不是這次對話內容出錯**——單純是這個 session 累積的「訊息則數」（不是 token 數，是離散的訊息/工具呼叫回合數）超過後端 API 的硬性上限（800 則）。這個限制存在於 Claude Code CLI / pi harness 呼叫的底層 API 這一層，跟模型本身、跟這次任務難不難無關。

**最容易踩到的情境**：像今天這種「大量小步驟高頻操作」的任務——例如用 Node.js CDP WebSocket 一步步操作瀏覽器（點擊→截圖→確認→再點擊）、或反覆執行 `reg query`/`sc query`/`Get-PnpDevice` 這類逐條下 bash 指令的除錯循環——每一輪工具呼叫 + 回應都會佔用訊息則數配額，很容易在還沒有「感覺話講很多」的狀況下，訊息則數就先破表。

## 處理方式

1. **這個 session 沒救了，訊息數量已經卡死上限，不會自己降下來**——不要嘗試重試同一個請求，格式正確的訊息一樣會被擋。
2. **必須開新的 session/新對話**才能繼續。Claude Code CLI 本身有 `/compact` 或類似的手動壓縮指令可以在訊息數還沒破表前主動觸發，但一旦已經跳出這個 413 錯誤，代表已經來不及在原 session 內壓縮了——只能靠外部機制（如 pi harness 的「compact 摘要後開新 session 接續」流程）把之前的進度轉寫成摘要，帶到新 session 繼續。
3. 如果是自己手動维护一個長時間跑的任務（例如今天這種「反覆測試—觀察—調整」的除錯循環），**主動注意訊息則數**，感覺已經跑了很多輪工具呼叫時提早收斂/開新 session，不要等到自動撞牆。

## 預防：減少高頻小步驟操作的訊息消耗

- **把多個獨立但相關的檢查合併成一次 bash 呼叫**（用 `&&`/`;` 串接，或寫一支腳本一次做完，而不是分好幾次工具呼叫個別執行），能省下大量訊息輪次。
- CDP/瀏覽器自動化這類「點一下就要看一次結果」的任務特別容易超標，能寫成一支完整腳本（做完整個流程再回報結果）就不要拆成一步一步呼叫再看螢幕截圖確認。
- 若預期任務本身就會需要非常多輪工具呼叫（例如系統性重複測試、大量除錯嘗試），可以事先跟使用者說明可能需要中途切一次新 session，把目前的結論/進度做個摘要交接，不要等到自動撞上 413 才處理。

## 相關

- 這個限制是 API 層級的固定值（800 則），跟哪個模型/哪個 profile 無關，換模型不會解決問题。
- 若使用 pi harness，session 之間的交接靠「compact 摘要」機制（把進度寫成結構化摘要帶到下一個 session），這也是本文件所在這個 skill 系統本身依賴的機制。

---

## Conformance Addendum

## When to Use
解讀並處理 Claude Code / pi harness 出現「Error 413 chat_history_too_large / payload_too_large」（訊息內容 "Chat history exceeds the 800-message limit; compact the conversation and retry."）錯誤。當使用者遇到這個錯誤訊息、或抱怨「對話卡住/送不出去/一直失敗」且懷疑是對話太長，或要求「處理 800 message limit」「chat history too large」時使用。

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
