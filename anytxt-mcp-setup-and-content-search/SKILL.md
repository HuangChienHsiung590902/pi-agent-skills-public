---
name: anytxt-mcp-setup-and-content-search
description: 啟用並使用 AnyTXT Searcher 的本機 JsonRPC API 搭配 MCP server，做大量檔案的全文內容搜尋、片段擷取與 OCR，並注意 token/context 用量控管。當使用者裝有 AnyTXT Searcher（ATGUI.exe）且想對大量本機檔案做內容層級搜尋（不只是檔名）時使用。
---

# AnyTXT MCP 設定與內容搜尋

## When to Use
當使用者裝有 AnyTXT Searcher（ATGUI.exe）且想透過 AI agent 對大量本機檔案做內容層級搜尋（不只是檔名），例如整理雜亂的磁碟、找出某個關鍵字出現在哪些文件、對圖片做離線 OCR。也適用於需要把 AnyTXT 的 7 個官方 JsonRPC 方法（Search/GetResult/GetFragment/GetFragmentAll/SyncIndex/GetRawTextByFID/OCR）包裝成 MCP 工具給 agent 使用的場景。

## Procedure
1. 確認 ATGUI.exe 是否在跑，並用 curl 測試 RPC 埠（預設 127.0.0.1:9920）是否可連通：POST 一個 Search 方法的 JsonRPC 請求，若 connection refused 代表服務未啟動
2. 若連不上，檢查 AnyTXT 的 config.db（SQLite，通常在 `<資料目錄>\Anytxt\config\config.db` 的 Setting 表）裡的 `HttpSearch` 欄位是否為 `'1'`。這個欄位控制 JsonRPC API 是否啟用；若為 `'0'`，需先關閉 ATGUI.exe 再用 sqlite 更新該值為 `'1'`（改資料庫前務必先備份 .db 檔），然後重新啟動 ATGUI.exe，日誌檔（ATGUI_*.log）會印出 `Started JsonRPC SDK service at 127.0.0.1:9920` 確認生效
3. 依序用 curl 實測 7 個方法驗證可用性：Search（回傳筆數）→GetResult（取得檔案清單與 FID，注意 output.files 是陣列的陣列，欄位順序由 output.field 給出）→GetFragment/GetFragmentAll（用 FID+pattern 取得比對片段）→GetRawTextByFID（取得整份檔案文字，可能很大）→SyncIndex（重新索引指定資料夾）→OCR（需要 OCR 版本才有效，圖片路徑建議用正斜線避免跳脫問題）
4. 若要包裝成 MCP server：實作時把 GetResult 回傳的陣列式 files 轉換成 `[{fid, lastModify, size, file}]` 物件陣列方便使用；GetRawTextByFID 務必加上預設截斷（例如 5000 字元）並在回應中標記 truncated/fullLength，避免單一大檔案的全文一次灌爆 agent 的 context window，同時保留可調的 maxChars 參數供需要時放寬
5. 額外加一個非官方的 status/診斷工具，內部呼叫一次輕量 Search 測試連通性，回報是否可達，並在失敗時提示檢查 HttpSearch 設定與 ATGUI.exe 是否在跑，方便除錯
6. 把 MCP server 加入 agent 的 mcp.json 設定（mcpServers 底下新增一個 entry，command 指向 node，args 指向 index.js），確認 JSON 格式正確（可用 python -m json.tool 或等效方式驗證）；若想暫時停用某個 MCP server 又不想刪除程式碼，直接把該 server 的區塊從 mcpServers 物件中移除即可，不用刪除實際檔案，之後隨時可以加回來

## Pitfalls
- 中文或含特殊字元的檔案路徑透過 bash 的 curl 測試時容易因雙重跳脫或編碼問題失敗（例如 OCR 回傳 invalid request），這通常是 bash/curl 傳遞層的問題不是 API 本身的問題，用英文路徑重測可驗證；若是用 Node.js 的 JSON.stringify 組請求則不會有這個問題，因為它會正確處理跳脫
- SyncIndex 的 folder 參數如果路徑含反斜線，在某些 shell（如 bash 下用雙引號包住 Windows 路徑）容易被吃掉變成連續字元（例如 `C:\Users\Desktop\` 變成 `C:UsersDesktop\`），建議路徑一律用正斜線，或透過程式語言的 JSON 序列化而非手動組字串
- GetRawTextByFID 沒有內建大小限制，對大型檔案（log、原始碼）直接呼叫可能一次回傳數萬字元，嚴重消耗 agent 的 context/token，應該預設截斷並提示改用 GetFragment/GetFragmentAll 只抓比對片段
- MCP server 本身即使定義了很多工具，只要沒被列在 agent 實際讀取的 mcp.json 的 mcpServers 裡，就完全不會佔用 token/context，「裝上但不用」和「沒裝」在 token 成本上幾乎沒差別，真正要注意的是「用的時候」某些方法（尤其 GetRawTextByFID/GetFragmentAll）回傳內容可能很大
- 同一台機器上可能存在多個看起來像是 MCP 設定的檔案（例如同時有 .claude/mcp.json、.config/mcp/mcp.json、.pi/agent/mcp.json），實際生效的是哪一個要先查證（讀取該 agent 執行環境的原始碼或用其內建的 MCP 管理 UI 確認），不要假設只有一處，也不要在沒確認清楚 loader 邏輯前，貿然把設定檔改成非標準副檔名（如 .jsonc）測試，這種測試可能導致該次對話所有 MCP 工具暫時消失，如果一定要測試，務必先完整備份原檔並準備好立即復原的步驟

## Verification
1. curl 對 127.0.0.1:9920 送出 Search 方法請求，回傳 JSON 帶有 result.data.output.count 而非 connection refused
2. 依序呼叫 GetResult → GetFragment/GetFragmentAll → GetRawTextByFID 都能對真實檔案的 FID 取得預期內容
3. 若包裝為 MCP server，用 `node --check index.js` 確認語法正確，並實際啟動 server 透過 stdio JSON-RPC 呼叫 tools/call 驗證每個工具回傳格式符合預期（包含 reshape 後的 GetResult 與截斷後的 GetRawTextByFID）
4. 把 server 加入 mcp.json 後，重啟該 agent/CLI，確認對應工具（如 anytxt_search 等）出現在可用工具列表中且能正常呼叫

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
