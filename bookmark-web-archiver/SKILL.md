---
name: bookmark-web-archiver
description: 將 Chrome 書籤匯出的 HTML 檔案，批次抓取每個書籤網址的網頁內容並轉成 Markdown 檔案，依照書籤資料夾結構鏡射輸出目錄。當使用者要求「整理書籤」「把書籤內容存成 md」「批次抓取書籤網頁內容」「書籤歸檔」「archive/backup bookmarks」時使用。內含可續跑（resume-safe）、依錯誤類型分類（登入牆/內網位址/非HTML/逾時）的 Python 抓取腳本，以及事後驗證抓取品質的稽核腳本。已知限制：僅用 requests（不執行 JavaScript），純 JS 渲染的 SPA 頁面與需要登入的頁面只會抓到殼子畫面而非實際內容，需搭配稽核腳本抽查。
---

# Chrome 書籤網頁內容歸檔

把一份 Chrome 匯出的書籤 HTML，批次轉成一堆 Markdown 檔案：每個書籤一篇，檔案內容是該網頁的正文擷取，輸出目錄結構鏡射原本的書籤資料夾階層。

## 什麼時候用

使用者手上有從 Chrome 匯出的書籤 HTML 檔，想把書籤網址的網頁內容存下來做離線備份/歸檔，而不只是存一份網址清單。

## 如何匯出 Chrome 書籤

1. Chrome 網址列輸入 `chrome://bookmarks/`
2. 右上角「⋮」→「匯出書籤」
3. 存成 `.html`（Netscape Bookmark File 格式，這是 Chrome/Firefox/Edge 共用的標準匯出格式）

## 相依套件

```bash
pip install requests trafilatura markdownify beautifulsoup4 lxml lxml_html_clean
```

（`lxml_html_clean` 是 trafilatura 的間接相依，新版 lxml 拆分出去了，沒裝會在 import 時炸掉，記得一起裝。）

## 使用方式

腳本在 `scripts/fetch_bookmarks.py`：

```bash
python scripts/fetch_bookmarks.py --source "書籤.html" --out-dir "輸出目錄"
python scripts/fetch_bookmarks.py --source X.html --out-dir Y --force      # 全部重抓覆蓋
python scripts/fetch_bookmarks.py --source X.html --out-dir Y --limit 10   # 先測試前 10 筆
python scripts/fetch_bookmarks.py --source X.html --out-dir Y --workers 12 # 調整平行數(預設8)
```

不帶 `--force` 時預設**略過已存在的輸出檔**，所以可以安全中斷（例如中途按 Ctrl+C 或程式當掉）、之後直接重跑同一個指令繼續補完，不會重抓已完成的部分。書籤量大（幾百筆以上）時建議先用 `--limit 10` 測試沒問題再跑全量。

## 運作原理

1. 用正則解析 Netscape 書籤 HTML（`<H3>` = 資料夾, `<A HREF>` = 書籤連結），重建資料夾路徑階層
2. 依「資料夾路徑 + 標題」產生輸出路徑（檔名做過危險字元清洗 `\/:*?"<>|`→`_`），輸出目錄結構鏡射書籤資料夾
3. 用 `requests` 抓網頁 → `trafilatura` 擷取正文轉 Markdown；擷取內容太短（<80字）時退回 `BeautifulSoup` 取 `<body>` 後用 `markdownify` 整頁轉換
4. 以下狀況會被辨識並寫入明確失敗原因，不會讓整支程式中斷：
   - 內網位址（`10.x` / `192.168.x` / `172.16-31.x` / `127.0.0.1` / `localhost`）
   - `chrome://` 等瀏覽器內部協定
   - 非 HTML 內容（PDF、圖片等）
   - HTTP 4xx/5xx 錯誤
   - 連線逾時 / DNS 失敗
5. 用 `ThreadPoolExecutor` 平行抓取（預設 8 個 worker），跑完印出成功/失敗/略過統計與失敗清單

## 已知限制（重要，务必先讓使用者知道）

`requests` 不會執行 JavaScript，所以：

- **登入牆頁面**（GitHub 需登入才能看的頁面、Hugging Face token 頁、opencode workspace 等）只會抓到登入表單/按鈕文字（如「Continue with GitHub」）
- **純 JS 渲染的 SPA**（GitHub 檔案樹頁、React/Vue 應用首頁）只會抓到「載入中」殼子畫面（如「There was an error while loading. Please reload this page.」）

這類頁面**不會**被腳本標記為失敗（因為技術上擷取到 >80 字的文字，通過門檻），是「偽成功」。實測 471 筆書籤的經驗值：明確失敗約 29%（內網位址/HTTP 錯誤，正確判斷）、偽成功約 3~5%、真正抓到實質內容約 65%。偽成功比例可接受，因為那些頁面本來就需要登入憑證才能取得真實內容，無法在無帳密的情況下自動化解決；若真的需要處理 JS 渲染頁面，要改用 Playwright/headless browser 才行（速度會慢很多，此腳本目前未整合）。

## 驗證抓取品質

跑完之後強烈建議用 `scripts/audit_archive.py` 稽核，不要假設「沒標記失敗 = 內容正確」：

```bash
python scripts/audit_archive.py --dir "輸出目錄"
```

會分成四類報告：
1. 明確失敗（含 `⚠️ 網頁抓取失敗` 標記）
2. 內容過短（<100 字，可疑）
3. 疑似偽成功（<1000 字且含「登入/Sign in/Loading/跳轉」等關鍵字）
4. 真正成功

並把「疑似偽成功」與「內容過短」清單印出來供人工抽查，這兩類通常集中在需要登入的私有系統或 SPA 頁面。

## 輸出格式

每篇書籤一個 `.md` 檔，檔名為 `{三位數流水號}_{清洗過的標題}.md`：

```markdown
# {標題}

- **原始網址 (URL):** {url}
- **書籤資料夾:** {folder_path}
- **抓取時間:** {yyyy-mm-dd hh:mm:ss}

---

{擷取到的 Markdown 內容，或失敗時顯示 "> ⚠️ 網頁抓取失敗: {原因}"}
```

## 書籤量很大時的建議流程

1. 先確認書籤總筆數（可以先跑一次 `parse_bookmarks()` 或直接 `--limit 5` 測試腳本可用）
2. 直接跑全量，`--workers` 依網路狀況調到 8~12
3. 跑完用 `scripts/audit_archive.py` 檢查偽成功清單
4. 對偽成功清單裡「內部系統/需要登入」的項目，如果使用者有帳密，可以手動補；否則直接接受標記失敗即可，不用勉強自動化

---

## Conformance Addendum

## When to Use
將 Chrome 書籤匯出的 HTML 檔案，批次抓取每個書籤網址的網頁內容並轉成 Markdown 檔案，依照書籤資料夾結構鏡射輸出目錄。當使用者要求「整理書籤」「把書籤內容存成 md」「批次抓取書籤網頁內容」「書籤歸檔」「archive/backup bookmarks」時使用。內含可續跑（resume-safe）、依錯誤類型分類（登入牆/內網位址/非HTML/逾時）的 Python 抓取腳本，以及事後驗證抓取品質的稽核腳本。已知限制：僅用 requests（不執行 JavaScript），純 JS 渲染的 SPA 頁面與需要登入的頁面只會抓到殼子畫面而非實際內容，需搭配稽核腳本抽查。

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
