---
name: pi-supermemory-auto-capture
description: "讓 Pi Agent 將長期有用的專案決策、使用者偏好、環境設定與修復結果自動保存到已設定的 Supermemory Local。只要對話中出現這四類資訊、使用者說『記住』、『存到 Supermemory』，或任務完成後產生可重用的決策與修復結論，就使用本 Skill；不要保存密碼、API token、私鑰或其他敏感憑證。"
compatibility: "需要 Pi MCP adapter 與 supermemory-local MCP bridge；Supermemory 服務位於 GPU Server 10.145.119.19:6767。"
---

# Pi Agent Supermemory 自動記憶

## When to Use

使用者在 Pi 對話中出現以下任一情況時載入並遵循本 Skill：

- 做出會影響後續工作的專案決策、架構選擇或操作約定。
- 使用者表達穩定的偏好、工作習慣或回覆方式要求。
- 確認了可重用的環境設定、服務位置、工具接線或版本資訊。
- 完成除錯、修復、部署或驗證，且結果對未來任務有參考價值。
- 使用者說「記住」、「存到 Supermemory」、「以後都這樣做」或類似要求。

不要把一般閒聊、一次性暫存資訊、完整對話逐字稿或敏感憑證當成長期記憶。

## Inputs and Outputs

### Inputs

- 本回合中已確認的決策、偏好、環境設定或修復結果。
- 可選的專案名稱、主機名稱、服務名稱或其他非敏感分類資訊。

### Outputs

- 透過 `supermemory_add` 寫入一筆精簡、可搜尋的長期記憶。
- 預設使用 `containerTag`：`pi-agent-auto-capture`；若明確屬於既有專案，可使用穩定且不含秘密的專案標籤。
- 回報是否已送入 Supermemory；`queued` 代表服務已接受，不能直接宣稱背景 indexing 已完成。

## Procedure

1. **判斷是否值得保存。** 只保存未來可能影響決策或減少重複排查的資訊，優先保存結論、原因、適用範圍與限制。
2. **先排除秘密。** 從內容移除密碼、API key、access token、私鑰、cookie、完整 Authorization header、個人敏感資料及可用來登入的連結。若內容主要是秘密，不要寫入。
3. **去重與整理。** 用一句到數句繁體中文寫成可獨立理解的記憶；必要時先使用 `supermemory_search` 查詢相近內容，避免反覆寫入相同事項。
4. **寫入。** 呼叫 `supermemory_add`：
   - `content`：包含「事項、結論、必要背景、限制」的摘要。
   - `containerTag`：預設 `pi-agent-auto-capture`，或使用已確認的非敏感專案標籤。
5. **確認結果。** 檢查工具回應的文件 ID 與狀態。若是 `queued`，說明已接受、仍由背景處理；若使用者要求立即確認，使用 `supermemory_get_document` 或 `supermemory_search` 驗證。
6. **不要過度保存。** 同一回合的多個細節若屬同一結論，合併成一筆；不要因每次工具呼叫都產生一筆記憶。
7. **使用記憶。** 後續遇到相同專案、主機、設定或偏好時，先用 `supermemory_search` 查詢相關內容，再將結果與目前實際狀態交叉確認；記憶可能過時，不得取代現場檢查。
8. **失敗時誠實回報。** 若 MCP bridge、GPU Server 或 Supermemory 不可用，不要假裝已保存；回報失敗原因並在本回合保留待重試的摘要。

## Rules and Limitations

- 永遠不要把秘密或可登入資料寫入 Supermemory，也不要在回覆中回顯秘密。
- 自動保存是「判斷後保存」而非逐字記錄所有 Pi 對話。
- 服務回傳 `queued` 僅表示已接受寫入；只有查詢到文件或狀態為 `done` 才能說已完成索引。
- Supermemory 記憶是參考資料；遇到主機狀態、服務埠號、版本與設定時，仍要以當下檢查結果為準。
- 不要為了保存記憶而中斷主要任務；主要任務失敗時，只有確定的修復結論才保存。
- 不要刪除、重建或清空 Supermemory 資料目錄來處理寫入問題，除非使用者另行明確授權。

## Pitfalls

- 把「已送入佇列」誤報成「已完成索引」。
- 把整段聊天、shell 輸出或環境檔原文直接存入，導致秘密外洩或記憶難以搜尋。
- 沒有帶一致的 `containerTag`，之後搜尋時誤以為記憶遺失。
- 把過時的記憶當成目前事實，未重新檢查就修改系統。
- 將 Supermemory 服務管理問題與本 Skill 的記憶保存行為混淆；服務安裝、啟動、修復請改用 `supermemory-local`。

## Verification

1. 確認 Pi MCP adapter 能列出 `supermemory_add`、`supermemory_search`、`supermemory_get_document` 與 `supermemory_health`。
2. 呼叫 `supermemory_health`，確認 GPU Server Supermemory 回應 HTTP `200`。
3. 寫入時取得文件 ID，且回應狀態至少為 `queued`。
4. 需要完整驗證時，用相同 `containerTag` 搜尋剛寫入的摘要，確認結果可取回；或輪詢文件直到 `done`／`failed`。
5. 若是修改 Skill 本身，執行 `D:\OB\skills\skill-creator\scripts\quick_validate.py D:\OB\skills\pi-supermemory-auto-capture`，並確認 `D:\OB\skills\SKILLS_INDEX.md` 已包含此 Skill。
