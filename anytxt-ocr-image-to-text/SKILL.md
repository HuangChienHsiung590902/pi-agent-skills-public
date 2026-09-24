---
name: anytxt-ocr-image-to-text
description: 使用 AnyTXT Searcher 內建離線 OCR 引擎，將圖片截圖辨識成文字，並將結果整理寫入 Obsidian 筆記庫（D:\OB\Inbox）。當使用者要求 OCR 一張/一批截圖、把截圖裡的文字抓出來、或要測試/驗證 AnyTXT OCR 功能時使用。
---

# AnyTXT OCR 圖片轉文字

## When to Use
當使用者要求「OCR 一張/一批截圖」、「把截圖裡的文字抓出來」、或要測試/驗證 AnyTXT OCR 功能時使用。前提是本機已安裝 AnyTXT Searcher 的 OCR 版本，且已透過 `anytxt-mcp-setup-and-content-search` skill 建立好 MCP 連線。

## Procedure
1. 先確認來源圖片實際位置：不要假設圖片在哪裡，先問清楚或用 find/PowerShell 實際搜尋確認。常見來源：`C:\Users\HCH\Pictures\Screenshots\`（Windows 預設截圖夾）、pi-clipboard 類型的臨時圖檔（使用者快照剪貼回傳）、或使用者指定的其他路徑
2. 若不確定 AnyTXT RPC 服務是否在線，先呼叫 anytxt_status 確認 rpcUrl (http://127.0.0.1:9920) 可達
3. 對每張圖片呼叫 anytxt_ocr，參數 file 傳入絕對路徑，優先使用正斜線 `/` 而非反斜線 `\`，避免 Windows 路徑因 backslash 跳脫字元導致解析錯誤（例如 `C:/Users/HCH/Pictures/Screenshots/xxx.png`）
4. 回傳格式為 `{ data: { output: { text: "..." } } }`，只有純文字，沒有座標/信心度資訊。若需要批量處理多張圖，可在同一個 tool call 回合中並發呼叫多次 anytxt_ocr（彼此獨立、無依賴關係）
5. **語義層矯正（由 LLM 執行，AnyTXT 本身不具備）**：AnyTXT 的 OCR 只是純字形辨識，沒有語義理解能力，看到「安装」不會知道繁體語境該是「安裝」。所以拿到原始 OCR 文字後，先由 LLM 自己讀懂上下文，把明顯不合理、疑似簡體殘留字形或同音錯字的地方修正過來（例如「安装」→「安裝」、「装程」→「安裝程」），這一步是語意判斷，不是查字典
6. **保留原始文字 + 標註修正**：輸出時務必同時列出「OCR 原始文字」與「語義矯正後文字」兩個版本，並列出具體改了哪幾處，不要只丟一個改過的版本讓使用者以為那是 OCR 原始結果；不要自行過度改寫語氣或補字，只修正明顯的辨識錯誤
7. **AnyTXT 全文搜尋交叉驗證（可選，用於專有名詞/人名/術語）**：對於語義判斷也拿不準的詞（尤其是專有名詞、人名、內部系統代號），可以呼叫 anytxt_search / anytxt_get_result 在使用者既有文件庫裡搜尋這個詞或字形相近的詞，若搜出來的既有文件裡有更常見、更合理的寫法，可作為佐證用來輔助判斷該怎麼修正；找不到就不要瞎猜，維持原樣並標注不確定
8. 將整理後的結果寫入 `D:\OB\Inbox\`（使用者的 Obsidian 筆記庫），檔名建議格式：`<主題>-<來源>-YYYYMMDD.md`，包含 YAML front matter（date/tags）、結論摘要、每張圖的「OCR 原始文字」與「語義矯正後文字」對照、來源檔名，以及使用方式備忘
9. 寫入前先確認 `D:\OB` 本身存在且是 Obsidian 庫（有 `.obsidian` 資料夾），避免寫錯路徑

## Pitfalls
- 不要假設使用者說的「截圖筆記」一定存在或位於特定資料夾，先實際搜尋（find / AnyTXT 全文搜尋）確認，若找不到就直接問清楚，不要捏造不存在的結果
- PowerShell 呼叫 `Get-ChildItem -Recurse` 搭配複雜 regex 的 `-notmatch` 或 `Where-Object` 時，若 pattern 包含多個 `|` 分隔的關鍵字，在經過 bash 單引號包裝後很容易因編碼/跳脫問題整個炸開產生巨量錯誤輸出（實測發生過，錯誤訊息重複輸出數百行）；這種情況下改用純 bash 的 find 指令更簡單可靠
- AnyTXT OCR 對中文的辨識準確度有限，常見錯誤：同音字誤判（安裝→安装）、連續數字中間漏字、原本就亂碼的畫面內容依舊會輸出亂碼（不會自動修正），不適合需要逐字正確的場合（如金額、序號），一定要提醒使用者人工複核
- AnyTXT 本身完全沒有語義判斷/錯字矯正能力，它只是全文索引/精確關鍵字比對引擎；不要誤以為呼叫 AnyTXT 的某個方法就能自動修正 OCR 錯字，矯正邏輯必須由 LLM 自己讀懂上下文後判斷，AnyTXT 頂多用來交叉驗證專有名詞在既有文件庫裡的寫法
- 語義矯正時不要對金額、序號、日期這類「看起來錯但無法從上下文推斷正確值」的內容自行腦補修正，這種字元錯誤只能標注可疑、交給使用者核對原圖，不能用語言模型的『合理猜測』取代真實數字
- file 參數使用反斜線（`C:\Users\...`）有時會因 JSON 字串跳脫問題失敗或讀到錯誤檔案，優先改用正斜線路徑
- 不要把 OCR 結果寫到可能被 Deep Freeze 凍結清空的 C 槽路徑，固定寫入 `D:\OB\Inbox\`（或使用者指定的其他 D 槽路徑）
- 若一次要處理大量圖片（十張以上），先與使用者確認範圍、不要一次性對整個大資料夾盲目 OCR，避免浪費時間與產出難以審核的巨量結果

## Verification
1. anytxt_status 回傳 reachable: true
2. 每筆 anytxt_ocr 呼叫回傳 errno: 0 且 output.text 非空
3. 寫入 `D:\OB\Inbox\` 的 .md 檔案能被 Read 工具正常讀取，且內容包含所有測試圖片的「OCR 原始文字」與「語義矯正後文字」對照、以及來源檔名
4. 人工目視比對至少一張圖片的原始畫面與 OCR 結果，確認大體內容符合（不需要逐字完全一致，但要確認沒有完全雞牙或空白輸出）
5. 檢查語義矯正後文字是否明確列出「改了哪幾處」，而不是靜默替換讓使用者無從得知哪些字被動過

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
