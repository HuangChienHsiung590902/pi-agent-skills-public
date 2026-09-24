---
name: nginx-demo-rich-menu-editor
description: 維護遠端 nginx demo 網站 https://demo.james-huang.org/ 的單頁圖文選單 UI；當使用者要求修改 demo.james-huang.org、nginx demo、方塊選單、LINE 圖文框、可拖放手機桌面式選單、多頁滑動、資料夾/子資料夾、自定義按鈕、或修該頁 JavaScript/CSS 互動時使用。
---

# nginx demo 圖文選單編修

## 任務目標

維護遠端主機 `hch@10.145.119.19` 上的 nginx 靜態 demo 頁：

- 公開網址：`https://demo.james-huang.org/`
- Cloudflare Tunnel：`/home/hch/nginx-demo-cloudflared/config.yml`
- Tunnel ingress：`demo.james-huang.org -> http://127.0.0.1:80`
- Docker 容器：`nginx`（`nginx:alpine`）
- 網站目錄：`/home/hch/nginx-demo-site`
- 首頁檔案：`/home/hch/nginx-demo-site/index.html`

目前頁面是一個純前端、無後端資料庫的手機桌面式圖文選單：

- 3 個可左右滑動的畫面。
- 每頁 `7 × 4`，共 28 格；總共 84 個預設入口。
- 方框大小需一致，配合 16:9 畫面鋪滿頁面。
- 每格右上角有淡色三點 `⋮` 自定義按鈕。
- 支援拖放排序、拖到其他格建立資料夾、拖到資料夾內、資料夾內再建立子資料夾。
- 子資料夾可展開、返回上一層，也可點選其中項目。
- 資料夾只剩 1 個項目時，應自動解除資料夾，該項目回到完整方框。
- 使用遠端 SQLite 儲存網站狀態；前端透過同源 `/api/state` 讀寫，`localStorage` 只作 API 暫時不可用時的 fallback。
- SQLite API 容器：`demo-state-api`，資料庫主機路徑 `/home/hch/nginx-demo-state/demo-state.sqlite3`，容器內 `/data/demo-state.sqlite3`。
- SQLite API：`GET /api/state` 讀取目前狀態、`PUT /api/state` 整體寫入狀態、`GET /api/health` 健康檢查；每次 PUT 同時更新 `app_state` 並寫入 `state_revisions`。
- 點選一般圖文格會用該格 `url` 的 hash 關鍵字（例如 `#payment` -> `payment`）呼叫 `openDynamicPage(keyword, item)`，開啟專屬動態頁；外部 `https://...` 連結仍直接跳轉。
- 動態頁左上角顯示麵包屑路徑，例如 `第 1 頁 / 資料夾 / 圖示名稱`；由 `findBreadcrumbByKeyword()` 遞迴搜尋多層資料夾。

## 何時使用

使用者提到以下任一情境時使用：

- `demo.james-huang.org`
- `nginx demo`、`demo nginx docker`
- 編修 demo 網站首頁或 `index.html`
- 九宮格、方塊選單、圖文選單、LINE 圖文框、Rich Menu
- 手機桌面式 UI、左右滑動多頁、頁面切換箭頭、底部圓點
- 拖放排序、方框放到方框、資料夾、子資料夾、返回上一層
- 三點自定義按鈕、淡色按鈕、hover cursor、滑鼠手指游標
- 該頁瀏覽器 console JavaScript 錯誤
- 修改後狀態必須寫進 SQLite，除非使用者明確執行 reset，不可只留在瀏覽器 localStorage
- 點選格子要開啟屬於該格的動態內容頁，或用 keyword/hash 產生頁面

## 輸入與輸出

**輸入：**

- 使用者要改的 UI 行為、文案、格子數、互動方式或錯誤訊息。
- 必要時可用使用者提供的截圖判斷視覺問題。

**輸出：**

- 已修改的 `/home/hch/nginx-demo-site/index.html`。
- 遠端備份檔路徑。
- 驗證結果：`nginx -t`、JS 語法檢查、公開網址 marker 確認。

## 參考資源

- `references/current-template.html`：目前可運作的三頁滑動圖文選單範本。當要重做、回復或大改 UI 時先讀這份。
- `scripts/deploy_index.py`：把本機 HTML 備份並部署到遠端 nginx demo 的腳本。
- `scripts/state_server.py`：SQLite 狀態 API 的參考實作；正式遠端目前由 `/home/hch/demo-state-server.py` bind mount 至 `demo-state-api` 容器執行。

相對路徑皆以本 Skill 目錄為基準，例如：

```powershell
python D:/OB/skills/nginx-demo-rich-menu-editor/scripts/deploy_index.py D:/OB/skills/nginx-demo-rich-menu-editor/references/current-template.html --marker "Demo"
```

## 執行流程

1. **確認目標**
   - 先確認這次要改的是 `https://demo.james-huang.org/`，不是其他 WebMCP 或 LINE 官方 Rich Menu。
   - 查遠端狀態：
     ```bash
     ssh -o BatchMode=yes hch@10.145.119.19 "hostname; docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}' | grep -i nginx"
     ```

2. **抓回現行檔案到本機編修**
   - 不要直接用遠端 heredoc 貼大型 HTML/JS，容易被 shell 引號、emoji、中文轉義弄壞。
   - 用 `scp` 抓回本機：
     ```bash
     scp -q hch@10.145.119.19:/home/hch/nginx-demo-site/index.html C:/Users/HCH/demo-menu-current.html
     ```

3. **修改 HTML/CSS/JS**
   - 大改時優先用 Python script 對本機檔案 patch，避免手動改壞 minified-ish 單行 CSS/JS。
   - 保持功能原則：
     - 方框尺寸一致。
     - 三點 `⋮` 自定義按鈕要淡，不要太搶眼。
     - 可點元素 hover 要 `cursor:pointer`。
     - 拖曳區域可維持 `cursor:grab`。
     - 只有一個子項目的資料夾要自動解除。
     - 多層資料夾要可展開、返回、點選項目。
     - 三頁滑動切換、邊緣自動換頁、底部圓點要保留，除非使用者要求移除。
     - 一般 item 點選要呼叫 `go(url,item)`；hash 型 url 進入 `openDynamicPage(keyword,item)`，不要只改 `location.hash` 而沒有畫面。
     - 目前左右換頁不使用箭頭按鈕；用全域 `mousemove` 偵測滑鼠移到極左/極右邊緣（目前 `edge=8`）時直接切頁，並用 `edgeSide` + `lastEdgeSwitch` 防止連續抖動。不要放置會遮住邊緣格子的透明熱區。

4. **確認 SQLite 狀態服務**
   - 檢查容器與 API：
     ```bash
     ssh -o BatchMode=yes hch@10.145.119.19 "docker ps --format 'table {{.Names}}	{{.Status}}	{{.Networks}}' | grep -E 'nginx|demo-state'; curl -sS http://127.0.0.1/api/health"
     ```
   - 若尚未部署，使用 `scripts/state_server.py` 建立 API 容器，並讓 nginx 與 `demo-state-api` 同在 `demo-state-net`；nginx `/api/` location proxy 到 `http://demo-state-api:8090/`。
   - DB 必須使用 named host bind directory `/home/hch/nginx-demo-state` 保存，不能只放容器 writable layer。

5. **本機 JS 語法檢查**
   - 從 HTML 抽出 `<script>` 後用 Node 檢查：
     ```bash
     python - <<'PY'
     from pathlib import Path
     html = Path('C:/Users/HCH/demo-menu-current.html').read_text(encoding='utf-8')
     js = html.split('<script>', 1)[1].split('</script>', 1)[0]
     Path('C:/Users/HCH/demo-menu-current.js').write_text(js, encoding='utf-8')
     PY
     node --check C:/Users/HCH/demo-menu-current.js
     ```

6. **部署前備份，然後上傳**
   - 可手動執行：
     ```bash
     ssh -o BatchMode=yes hch@10.145.119.19 "cp /home/hch/nginx-demo-site/index.html /home/hch/nginx-demo-site/index.html.bak.$(date +%Y%m%d-%H%M%S)"
     scp -q C:/Users/HCH/demo-menu-current.html hch@10.145.119.19:/home/hch/nginx-demo-site/index.html
     ```
   - 或使用腳本：
     ```bash
     python D:/OB/skills/nginx-demo-rich-menu-editor/scripts/deploy_index.py C:/Users/HCH/demo-menu-current.html --marker "Demo"
     ```

7. **驗證**
   - nginx 語法：
     ```bash
     ssh -o BatchMode=yes hch@10.145.119.19 "docker exec nginx sh -c 'nginx -t'"
     ```
   - 公開站 marker：
     ```bash
     curl -k -s --max-time 15 https://demo.james-huang.org/ | grep -Eo 'Demo|PAGE_COUNT|renderPager|openChildFolder|page-arrow' | head
     ```
   - 若瀏覽器可用，優先用 Playwright 或人工瀏覽確認 console 無錯誤、滑動/拖放/展開可用。

## SQLite 持久化規則

- 前端啟動時先 `GET /api/state`，API 狀態為準；沒有資料時才用舊 localStorage/預設值初始化並 PUT 到 SQLite。
- 每次會改變網站狀態的操作都必須呼叫 `save()`：自定義儲存、拖放/建立資料夾、移出項目、頁面切換、reset。
- `save()` 以 `stateWriteChain` 序列化 PUT，避免快速連續操作時後一次寫入被前一次覆蓋。
- 狀態寫入使用 SQLite `app_state` 單列 current state；`state_revisions` 保留每次 PUT 的歷史版本。
- 重啟 `nginx` 或 `demo-state-api` 不會遺失狀態，因為 DB 在 `/home/hch/nginx-demo-state`。
- 使用者要求 reset 時才可用預設值覆蓋 SQLite；一般部署不能自動 reset。
- **安全提醒：** 目前 `/api/state` 是公開同源 PUT，任何能存取 demo 網址的訪客理論上都能修改共享狀態。若要只有管理者能修改，下一步必須加 Cloudflare Access、登入驗證或獨立管理端點；不能把秘密 token 直接硬編碼在公開 HTML。

## 規則與限制

- 這是純靜態前端頁；沒有後端，所以自定義資料存在每個瀏覽器自己的 `localStorage`。
- 若改 `localStorage` key，使用者瀏覽器會載入新版預設資料，但舊自定義可能不會延續；改 key 前要有理由。
- 不要重啟 nginx，除非修改 nginx config；目前 `index.html` 是 bind mount，改檔即生效。
- SQLite API 或 nginx `/api/` proxy 設定變更時才需要重啟相關容器；重啟前先確認 DB bind mount 存在。
- 不要刪除 `/home/hch/nginx-demo-site/images/` 的圖片。
- 不要執行破壞性 Docker 操作，例如刪容器、清 volume、prune，除非使用者明確要求並確認。
- 如果要修改 Cloudflare Tunnel 或 nginx container 參數，另讀 `docker-remote-control` skill。

## 已知坑

- **遠端 heredoc 會破壞 HTML/JS：** 大型 HTML 含中文、emoji、template literal、引號時，不要直接 `ssh "cat > file <<EOF"`；應先寫本機檔案再 `scp`。
- **JS 初始化順序：** 不要在 `let slots = load()` 初始化期間讓 `load()` 直接存取 `slots`，會報 `Cannot access 'slots' before initialization`。應用 `normalizeList(list)` 回傳資料，再賦值給 `slots`。
- **資料夾只剩一個項目：** 必須 normalize，自動解除資料夾，避免外層顯示小型資料夾預覽。
- **子資料夾：** 不能只 alert；要用 path stack（例如 `openFolderPath`）進入下一層，並提供返回上一層。
- **格子尺寸：** 外層與資料夾內要有固定 grid rows/columns 與 `width:100%; height:100%`，避免項目數不同導致忽大忽小。
- **游標與切頁箭頭：** `.edit-btn`、`.hint`、`.btn`、`.dot`、實際左右箭頭按鈕要 `cursor:pointer`；方框本體可 `cursor:grab`。左右換頁改為滑鼠移到極左/極右邊緣自動切頁；不要使用 `.page-zone` / `.page-arrow` 透明覆蓋層，以免遮住邊緣格子。

## 驗證清單

完成每次修改後至少確認：

1. 遠端有建立 `index.html.bak.<timestamp>` 備份。
2. `node --check` 對抽出的 JS 通過。
3. `docker exec nginx sh -c 'nginx -t'` 通過。
4. `curl -k -s https://demo.james-huang.org/` 可找到新版 marker。
5. `curl -k -s https://demo.james-huang.org/api/health` 回傳 `ok: true`。
6. `GET /api/state` 顯示 3 頁、每頁 28 格；PUT 測試後確認 SQLite `app_state` 有 1 筆且 `state_revisions` 有版本紀錄。
7. 若是互動修正，至少檢查相關關鍵字已出現在公開站 HTML，例如：
   - 多頁：`PAGE_COUNT=3`、`renderPager`、`touchend`
   - 邊緣換頁：`edgeSide`、`lastEdgeSwitch`、`edge=8`；確認沒有 `.page-zone` / `.page-arrow` 遮罩
   - 子資料夾：`openChildFolder`、`folderByPath`、`backFolder`
   - 動態頁：`detailPanel`、`detailBreadcrumb`、`keywordFromUrl`、`findBreadcrumbByKeyword`、`openDynamicPage`、`返回選單`
   - 單項資料夾解除：`children.length===1` 或 `normalizeList`

## Pitfalls

- 不要在沒有備份 `index.html` 與目前 SQLite 狀態的情況下直接部署。
- 不要把 `localStorage` 的 fallback 當成永久保存；狀態修改必須確認已寫入 `/api/state`。
- 不要未經使用者明確授權就刪除、reset 或覆蓋遠端 SQLite。
- 不要只驗證本機檔案；部署後還要確認 nginx、公開網址與狀態 API。
