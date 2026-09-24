---
name: "aile-desktop-cdp-chat-export"
description: "透過 Chrome DevTools Protocol 連接 Aile Desktop（Electron App），即時抓取完整聊天室歷史紀錄並匯出成 xlsx"
version: 1
created: "2026-08-13"
updated: "2026-08-13"
---
## When to Use
當使用者要求「連上 Aile Desktop」「讀取 Aile 聊天紀錄」「匯出某聊天室完整對話」時使用。適用於任何以 Electron 開發、支援 --remote-debugging-port 啟動參數的桌面應用程式，不限於 Aile。目標是取得比本機 IndexedDB 快取或舊 xlsx 匯出檔更即時、更完整的聊天歷史（因為使用者可能持續在用，本機快取不是最新的）。

## Procedure
1. 1. 確認目標 App 是 Electron 開發：檢查安裝路徑下有無 resources/app.asar，或進程指令列含有 --type=renderer/--type=gpu-process 等 Chromium 旗標。
2. 2. 找到執行檔路徑：用 `Get-CimInstance Win32_Process -Filter "Name = '<App>.exe'"` 或 `Get-Process` 取得 exe 完整路徑（例如 C:\Users\<user>\AppData\Local\Programs\<app>\<App>.exe）。
3. 3. 關閉現有進程再以除錯埠重啟：`Get-Process '<App>' | Stop-Process -Force`，等待2-3秒確認全部進程結束，再 `Start-Process -FilePath $exe -ArgumentList '--remote-debugging-port=9222'`。務必先取得使用者明確同意，因為這會登出/關閉使用者正在使用的視窗。
4. 4. 等待5-6秒讓 App 完全啟動，用 `curl -s http://localhost:9222/json/list` 確認有 type=page 的分頁出現，且 url 是 index.html（不是子視窗如 imgview 或登入 webview）。
5. 5. 用 Playwright browser_tabs (action:list/select) 切到正確的分頁（index.html 那個，不是 imgview 那個）。
6. 6. 用 browser_snapshot 確認畫面狀態：可能需要使用者本人在 App 視窗手動完成登入（手機驗證碼、帳密等），這一步AI不能代勞，也不要嘗試用 location.hash 硬導航，那樣可能觸發登出。
7. 7. 登入完成、看到聊天室列表後，用 browser_click 點擊目標聊天室，進入對話畫面。
8. 8. 找到訊息容器與『載入更多訊息』按鈕的 CSS class（一般是 .loadmorebtn），以及外層可捲動容器（PerfectScrollbar 常見 class 為 .ps.ps--active-y）。
9. 9. 用 browser_evaluate 執行 async 迴圈：反覆將 scroller.scrollTop = 0 並 dispatch scroll 事件，接著點擊 loadmorebtn（若存在），每輪間隔 350-550ms，直到連續 N 次（建議8-10次）偵測不到 scrollHeight 變化且按鈕消失才停止，並用全域變數如 window.__loadDone = true 標記完成，方便之後輪詢查詢。
10. 10. 這個載入迴圈執行時間可能長達20-30分鐘（取決於歷史訊息量），browser_evaluate 呼叫本身容易在30-60秒逾時，屬於正常現象——迴圈是在頁面內以 async 方式背景執行，不受 RPC 逾時影響，之後可用新的 browser_evaluate 呼叫查詢 window.__loadDone 與 scroller.scrollHeight 判斷進度，反覆 sleep 60-150秒後再查詢即可監控進度直到 done:true。
11. 11. 載入完成後，用 browser_evaluate 讀取訊息容器（例如 .mt-3.isUS 或類似 class）的所有直接子元素，逐一判斷：日期分隔線（純文字符合 /^\d{4}年\d{1,2}月\d{1,2}日$/ 且子元素極少）、系統訊息（含 .message_sys_2_content）、一般訊息（含 .message_name/.message_time_2/.message_content），組成結構化陣列 {date, type, sender, time, content}，並用 JSON.stringify 回傳。
12. 12. 若資料量大導致 MCP 回傳內容被截斷並存成暫存 txt 檔，用 Read 工具讀取該暫存檔路徑；由於暫存檔內容格式是 '### Result\n"<JSON字串>"\n### Ran Playwright code...'，需要用 Python 先定位 '### Result\n' 到 '\n### Ran Playwright code' 之間的內容，該內容是一個JSON字串常值（外層有引號跳脫），先 json.loads 解碼一次拿到純JSON字串，再 json.loads 一次才是真正的資料陣列。切勿直接用 bash cat/grep 顯示中文內容做判斷，終端機編碼（如 Windows cp950）會讓中文顯示成亂碼，但不影響檔案實際內容——一律用 python -X utf8 讀寫確認。
13. 13. 用 openpyxl 建立 Workbook，欄位建議：序號、日期、時間、類型、發送者、內容，設定欄寬與 wrap_text，儲存到指定路徑（例如 D:\<聊天室名稱>_完整對話紀錄.xlsx）。
14. 14. 用 python -X utf8 重新讀取存好的 xlsx 頭尾幾筆資料做驗證，確認筆數、日期範圍、內容正確。

## Pitfalls
- 不要用 `$` 前綴的 PowerShell 變數直接內嵌在 bash 的 -Command 字串裡執行，bash 會把 $var 當作自己的變數展開導致指令錯誤且可能重複執行上百次；一律把 PowerShell 腳本寫到暫存 .ps1 檔（heredoc `cat > file.ps1 << 'EOF' ... EOF`）再用 `powershell -NoProfile -ExecutionPolicy Bypass -File file.ps1` 執行。
- 不要用 `location.hash = '#/'` 或直接改網址硬導航 Electron SPA 到其他路由，這可能被 App 判定成需要重新驗證而觸發登出，讓使用者被迫重新登入。應該讓使用者留在 App 原生UI裡手動導航，AI 只在旁讀取/點擊既有可見元素。
- CDP 的 /json/list 有時只列出子視窗（如圖片檢視器 imgview）看不到主視窗，這通常是因為主視窗還在啟動中或被其他視窗遮住；不影響 CDP 連線本身，多等待幾秒重新查詢通常就會出現 index.html 的主分頁。視窗是否可見（前景/背景/被遮住）不影響 CDP 操作能力，不需要刻意把視窗帶到前景。
- browser_evaluate 呼叫長時間執行的 async 迴圈時，工具本身有請求逾時（約30-60秒），逾時只代表『這次RPC呼叫沒等到回應』，不代表頁面內的 JS 迴圈中斷了——迴圈仍在瀏覽器頁面內繼續跑。之後應該用新的 browser_evaluate 呼叫去『查詢』先前迴圈設定的全域旗標變數（如 window.__loadDone）與可觀察的進度指標（如 scrollHeight），而不是重新啟動一個迴圈，否則會有多個迴圈同時搶著點同一個按鈕。
- 虛擬滾動/分頁載入的訊息列表，只點擊『載入更多』按鈕本身可能不夠，還需要同時把捲動容器的 scrollTop 設為 0 並手動 dispatch scroll 事件，否則框架不會觸發下一批載入（尤其用了 PerfectScrollbar 這類套件時，scrollTop 賦值不一定會自動觸發原生 scroll 事件）。
- MCP 工具回傳大字串（例如完整訊息 JSON）超過約 300KB 時會被截斷並存成暫存檔，暫存檔內容不是純資料，而是整個工具呼叫記錄的文字轉錄（含 '### Result' 等標記），直接當作JSON解析會失敗；需要先用字串定位切出真正的 JSON 字串常值部分，再做兩層 json.loads解碼（因為回傳值本身是 JSON.stringify 過的字串，這個字串又被包在MCP的文字輸出格式裡）。
- Windows 終端機（尤其 git-bash/PowerShell 混用時）預設輸出編碼常是 cp950 或 Big5，直接用 bash echo/cat 顯示中文檔名或中文內容容易顯示成亂碼；這只是『顯示層』問題，不代表檔案本身壞掉。驗證中文內容正確性一律改用 `python -X utf8` 執行讀取與 print，不要以終端顯示的亂碼判斷資料是否正確。
- 重新啟動一個已登入的桌面 App 並加上 --remote-debugging-port 參數前，務必先明確告知使用者『這會關閉並重開視窗，可能需要重新登入』並取得同意，這是有感知的破壞性操作（會中斷使用者正在進行的工作），不應該未經確認就執行。

## Verification
1. curl http://localhost:9222/json/list 回傳的陣列裡包含 type:page 且 url 為該 App 的 index.html（非子視窗）。
2. browser_evaluate 查詢 window.__loadDone 為 true，且 scroller.scrollHeight 連續兩次查詢間不再顯著成長。
3. 解析出的訊息陣列筆數與畫面上看到的聊天室訊息量級相符（可用聊天室列表顯示的最早日期/最新日期做粗略比對）。
4. 存好的 xlsx 用 python -X utf8 + openpyxl 重新讀取，頭幾筆與尾幾筆的日期、發送者、內容中文字顯示正常、無亂碼，且總列數等於訊息陣列筆數。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
