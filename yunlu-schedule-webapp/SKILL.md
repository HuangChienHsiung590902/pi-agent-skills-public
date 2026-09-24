---
name: yunlu-schedule-webapp
description: 維護韻綠大樓排班系統本機與公開 Docker 版；當使用者要修改 C:\Users\HCH\Downloads\排班系統、排班規則、SQLite、Excel 匯出、UI 樣式、LINE Login、權限申請管理、Cloudflare Tunnel 或遠端 Docker 部署時使用。
---
# 韻綠大樓排班系統維護 Skill

## 任務目標

維護使用者的韻綠大樓排班 Web App。

系統用途是替韻綠大樓產生每月排班，並匯出與原始 Excel 班表格式一致的 `.xlsx`。目前同時有：

- 本機版：位於 `C:\Users\HCH\Downloads\排班系統`
- 公開版：部署在 Jetson Docker，透過 Cloudflare Tunnel 對外提供 `https://schedule.james-huang.org`
- 登入保護：LINE Login + SQLite 權限申請／核准流程

## 適用時機

使用者提到以下任一情境時載入本 Skill：

- `排班系統`、`韻綠大樓排班系統`
- 修改排班 Web 畫面、UI、按鈕、圖例、衝突提示、表格樣式
- 做四休一、第一個休假日、起始天數、跨月延續
- 廖欽得、蔡朋育、黃正鼎、楊榮聲排班規則
- SQLite 儲存、載入已存月份、刪除月份、逾期鎖定
- 匯出原格式 Excel、template.xlsx、Excel 版面與原檔一致
- `開啟排班系統.bat` 不能開、port 8766、舊伺服器佔用
- Docker 遠端部署、Cloudflare Tunnel、`schedule.james-huang.org`
- LINE Login、Channel ID / Callback URL、登入頁、登出、白名單、權限申請、管理員核准與 LINE 登入登出記錄表

## 系統架構

### 本機檔案角色

| 檔案 | 角色 |
|---|---|
| `C:\Users\HCH\Downloads\排班系統\index.html` | 前端 UI、排班計算、Excel 匯出邏輯 |
| `C:\Users\HCH\Downloads\排班系統\serve.js` | Node.js HTTP server、SQLite API、LINE Login、權限管理 |
| `C:\Users\HCH\Downloads\排班系統\schedule.db` | 儲存每個月份排班 JSON、登入記錄、權限申請、核准名單 |
| `C:\Users\HCH\Downloads\排班系統\template.xlsx` | 原始 Excel 格式模板；匯出時只替換年月、日期、星期與排班格 |
| `C:\Users\HCH\Downloads\排班系統\jszip.min.js` | 前端重新封裝 `.xlsx` 用 |
| `C:\Users\HCH\Downloads\排班系統\開啟排班系統.bat` | 啟動／重啟本機服務並開瀏覽器 |

### 本機版服務資訊

- URL：`http://127.0.0.1:8766/`
- 啟動：`C:\Users\HCH\Downloads\排班系統\開啟排班系統.bat`
- 本機 server 監聽：`PORT=8766`

### 公開 Docker 版服務資訊

- URL：`https://schedule.james-huang.org`
- 遠端主機：`ssh hch@10.145.119.12`
- 遠端程式目錄：`/home/hch/docker-webs/yunlu-schedule`
- Docker container：`yunlu-schedule`
- Docker image：`yunlu-schedule:latest`
- Origin 綁定：`127.0.0.1:18766 -> container 3000`
- Cloudflare Tunnel ID：`1ec91570-0cd8-4ce2-908a-3a1713de3114`
- Cloudflare 設定：`/etc/cloudflared/config.yml`
- Ingress rule：

```yaml
- hostname: schedule.james-huang.org
  service: http://127.0.0.1:18766
```

公開版 Docker port 應只綁定 `127.0.0.1:18766`，不要綁 `0.0.0.0`，避免繞過 Cloudflare Tunnel。

### API

- `GET /health`：服務健康檢查與 auth 設定狀態
- `GET /api/me`：目前登入者資料與是否管理員
- `GET /api/login-events`：管理員取得最近 LINE 登入／登出記錄，最新在最上面
- `GET /api/schedules`：列出已存月份
- `GET /api/schedule?month=YYYY-MM`：取得某月資料
- `POST /api/schedule`：儲存某月資料
- `DELETE /api/schedule?month=YYYY-MM`：刪除某月資料
- `GET /login`：LINE Login 登入頁
- `GET /auth/line`：導向 LINE OAuth
- `GET /auth/line/callback`：LINE OAuth callback
- `GET /logout`：清除登入 session
- `GET|POST /access/request`：未授權使用者送出權限申請
- `GET /admin/access`：管理員權限申請管理頁
- `POST /admin/access/:userId/approve`：核准申請
- `POST /admin/access/:userId/reject`：拒絕申請
- `POST /admin/access/:userId/revoke`：移除已核准使用者
- `POST /admin/access/demo-applicant`：建立測試申請者，供管理員預覽申請流程

> 若新增管理端操作，例如修改／刪除申請者，應繼續使用 `/admin/access/:userId/<action>` 的 POST 型式，並只允許管理員呼叫。

## 排班規則

### 人員與班別

| 人員 | 預設角色 |
|---|---|
| 廖欽得 | 固定 `A` 早班 |
| 蔡朋育 | 固定 `B` 晚班 |
| 黃正鼎 | 機動補班；廖欽得休補 `A`，蔡朋育休補 `B` |
| 楊榮聲 | `D` 清潔班 |

### 固定班週期

- 廖欽得與蔡朋育採「做四休一」。
- 表格內「休假」欄的下拉選單代表：**本月第一個休假日是幾號**。
  - `1號休` → 1、6、11、16、21、26 號休
  - `2號休` → 2、7、12、17、22、27 號休
  - 以此類推。
- 不要再把此欄解釋成「週期第幾天」。使用者曾明確修正這點。
- 若兩人的第一個休假日相同，系統需提示可能造成黃正鼎同日需補 A/B 衝突。

### 跨月延續

若切換到尚未儲存的新月份：

1. 先找前一個月份是否存在於 SQLite。
2. 若存在，依前月第一個休假日與前月天數推算本月第一個休假日。
3. 推算公式概念：下一月第一個休假日要延續五日週期，而不是每月固定同一天。

### 逾期鎖定

- 早於目前年月的月份視為逾期。
- 逾期月份只能查看、列印、匯出 Excel、刪除已存月份。
- 逾期月份不可修改排班、不可調整休假下拉、不可清除異動。

### 已存月份下拉同步

- `已存月份` 下拉選單顯示格式應只包含年月，例如 `2026年8月`。
- 不要在下拉文字中顯示更新時間或日期。
- 使用者選擇已存月份時，必須同步：
  - 年度欄位
  - 月份欄位
  - 隱藏 `#month` 的 `YYYY-MM` 值
  - 班表日期列
  - 該月份對應的排班資料
  - 狀態文字
- 從年度／月份欄切換到某個已存月份時，`已存月份` 下拉也應自動選中相同月份。

## UI 與設計慣例

目前 UI 使用 `frontend-design` 調整過，預設方向：

- 柔和、圓角、友善、深綠主色
- Header 左側使用大樓 SVG logo，來源概念是韻綠大樓照片
- Header 高度要緊湊，避免佔用太多垂直空間
- 排班主畫面應盡量在桌面 1440×800 一頁內顯示完整，不要需要垂直捲動
- 上班統計放左邊，使用一張大卡片顯示所有數據；提示放右邊
- 登入頁 logo 要與 Header 大樓 icon 一致，不要用臨時方框符號
- 登入後右上角要顯示 LINE 使用者資訊與 `登出` 按鈕
- 管理員登入後工具列要顯示 `申請名單`
- 主畫面工具列不要顯示 `SQLite` 技術標籤；使用者介面不應暴露底層儲存技術
- `/admin/access` 的申請者每一行右側直接顯示 icon 操作，不使用大文字按鈕：`✓` 同意、`✕` 拒絕、`🗑` 刪除

## Excel 匯出規則

- 使用 `template.xlsx` 作為唯一正式模板。
- 匯出時保留原始格式：合併儲存格、欄寬、列高、框線、字型、簽核區與列印設定。
- 只更新：
  - 年度／月份
  - 日期列
  - 星期列
  - 廖欽得、蔡朋育、黃正鼎、楊榮聲的排班格
- 注意原始 Excel 有些空白格是 `<c .../>`，有些有內容是 `<c ...>...</c>`；修改 xlsx XML 時兩種都要支援。
- 若移除 `xl/calcChain.xml`，也要同步移除 `[Content_Types].xml` 與 `xl/_rels/workbook.xml.rels` 的相關參照，避免 Excel 修復提示。

### 衝突紅字規則

- Web 表格中黃正鼎補班若同日需補 A/B 會顯示 `衝突`。
- Excel 匯出時，黃正鼎補班列若輸出 `衝突`，字型必須套用紅字。
- 不要直接改 `template.xlsx`；目前做法是在匯出時以 JSZip 修改 `xl/styles.xml`，基於原樣式動態新增紅字 style，再套到衝突儲存格。
- 曾確認 `template.xlsx` 內有紅字字型可複用；但實作仍應以目前模板的 `styles.xml` 為準，不要寫死未驗證的 style 結構。

## LINE Login 與權限申請

### LINE Login 設定

公開版使用 LINE Login 保護：

- Callback URL 必須登記：`https://schedule.james-huang.org/auth/line/callback`
- 目前 Channel ID：`2011263914`
- Channel Secret 只存在 Docker 環境變數，**不可輸出到回覆或寫入前端**。
- `ALLOWED_LINE_USER_IDS` 只放固定管理員名單。
- OAuth scopes：`profile openid`

Cookie：

- `yunlu_session`：已授權使用者登入 session，HttpOnly、SameSite=Lax、HTTPS 下 Secure
- `yunlu_oauth`：OAuth state / nonce 暫存
- `yunlu_applicant`：未授權但已完成 LINE 身分驗證的申請者暫存

重要實作細節：callback 成功時可能需要同時設定多個 `Set-Cookie`；`setCookie()` 必須保留既有 header，不能讓後設定的 cookie 覆蓋前一個。曾發生 `yunlu_oauth` 清除 cookie 覆蓋 `yunlu_session`，導致 LINE 驗證成功後仍回到登入頁。

### 白名單與管理員

- Docker 環境變數 `ALLOWED_LINE_USER_IDS` 是固定系統管理員名單。
- 目前管理員 LINE user id：`U6efbfa52ff5e78157672bc3fb9280ece`。
- 管理員可進入 `/admin/access` 查看與處理申請。
- 管理員登入排班主畫面後，工具列會顯示 `申請名單`。
- 頁面右上角有 LINE 使用者資訊與 `登出`；登出會清除 `yunlu_session` 與 `yunlu_applicant`。

### 權限申請流程

1. 使用者按 LINE Login。
2. LINE 驗證成功後取得 `userId`、`displayName`、`pictureUrl`。
3. 若 `userId` 在 `ALLOWED_LINE_USER_IDS` 或 `access_grants`，直接進入排班系統。
4. 若沒有權限，導向 `/access/request`。
5. 使用者可填寫申請原因，送出後寫入 `access_requests`，狀態為 `pending`。
6. 管理員在 `/admin/access` 可核准、拒絕或刪除申請者。
7. 申請者列表每一行右側使用 icon 操作：`✓` 同意、`✕` 拒絕、`🗑` 刪除。
8. 核准後寫入 `access_grants`；該使用者重新登入即可進入系統。

### 權限相關 SQLite 表

- `login_events`：LINE 登入／登出記錄。欄位包含 `line_user_id`、`display_name`、`event_type`、`event_at`；舊資料可能還有 `logged_in_at`，migration 時要相容。
- `access_requests`：登入權限申請。
- `access_grants`：管理員核准後的授權使用者。

### LINE 登入登出記錄表

- 管理頁 `/admin/access` 下方顯示 `LINE 登入登出記錄`。
- 記錄來源為 SQLite `login_events` table。
- 登入成功時寫入 `event_type='login'`。
- 使用者按 `/logout` 時寫入 `event_type='logout'`；若是申請者 cookie `yunlu_applicant` 登出，也要記錄。
- 顯示排序必須是最新在最上面：`ORDER BY datetime(event_at) DESC, id DESC`。
- 建議 UI 欄位：動作、使用者、LINE User ID、時間。
- 此表只給管理員看；API `/api/login-events` 必須經過 `requireAdmin(..., true)`。

### 測試未授權申請頁

因使用者目前主要只有一個有權限帳號，管理頁提供「模擬未授權使用者申請」按鈕：

- 入口：`/admin/access`
- 測試申請者 ID：`Udemo-access-request-preview`
- 用途：讓管理員預覽未授權者的申請頁、測試核准／拒絕／修改／刪除流程。

## 公開版部署流程

### 將本機檔案同步到 Jetson

```bash
scp 'C:/Users/HCH/Downloads/排班系統/index.html' 'C:/Users/HCH/Downloads/排班系統/serve.js' hch@10.145.119.12:/home/hch/docker-webs/yunlu-schedule/
```

視需求同步其他資源：

```bash
scp 'C:/Users/HCH/Downloads/排班系統/template.xlsx' 'C:/Users/HCH/Downloads/排班系統/jszip.min.js' hch@10.145.119.12:/home/hch/docker-webs/yunlu-schedule/
```

### 重建 Docker，保留機密環境變數與 SQLite

```bash
ssh hch@10.145.119.12 'set -e
APP=/home/hch/docker-webs/yunlu-schedule
docker inspect yunlu-schedule --format "{{range .Config.Env}}{{println .}}{{end}}" | grep -E "^(BASE_URL|LINE_CHANNEL_ID|LINE_CHANNEL_SECRET|SESSION_SECRET|ALLOWED_LINE_USER_IDS|NO_OPEN)=" > /tmp/yunlu-current.env
chmod 600 /tmp/yunlu-current.env
docker build -t yunlu-schedule:latest "$APP" >/tmp/yunlu-build.log
docker rm -f yunlu-schedule >/dev/null
docker run -d --name yunlu-schedule --restart unless-stopped -p 127.0.0.1:18766:3000 --env-file /tmp/yunlu-current.env -v "$APP/schedule.db:/app/schedule.db" yunlu-schedule:latest >/dev/null
rm -f /tmp/yunlu-current.env
for i in $(seq 1 30); do curl -fsS http://127.0.0.1:18766/health && break; sleep 1; done
'
```

### 重要部署約束

- Docker image 必須用 Node 24 以上，因 `serve.js` 使用 `node:sqlite`。
- 已確認 `node:20-alpine` 會報：`ERR_UNKNOWN_BUILTIN_MODULE: No such built-in module: node:sqlite`。
- `Dockerfile` 應使用：`FROM node:24-alpine`。
- `schedule.db` 以 volume 掛載：`/home/hch/docker-webs/yunlu-schedule/schedule.db:/app/schedule.db`，避免重建 image 時覆蓋資料。
- Cloudflare Tunnel 只連到 `127.0.0.1:18766`，不要把 Docker port 公開綁到 `0.0.0.0`。

## 常見問題與處理

### bat 不能開或刪除 API 無效

常見原因是舊版 Node server 還佔用 port `8766`。目前 `開啟排班系統.bat` 應先關閉舊的 `8766` listener，再啟動 `serve.js`。

檢查：

```bash
curl -s http://127.0.0.1:8766/api/schedules
```

應回傳 JSON，例如：

```json
{"ok":true,"items":[]}
```

若 DELETE 仍回 `404 Not Found`，代表瀏覽器或 port 還連到舊版 server，需要重啟 bat 或手動停止舊 process。

### SQLite 顯示「伺服器回應格式錯誤」

通常是 API 回傳 `Not found` 純文字，而不是 JSON。優先檢查目前 server 是否為新版 `serve.js`。

### 衝突樣式

- 表格內 `td.conflict` 才可以有動畫。
- 圖例 `.legend .conflict` 不應閃爍。
- 動畫要限制在格子內部，不要用外擴 box-shadow 造成超出格線。

### 年度不能改

不要使用瀏覽器原生 `type="month"` 作為主要 UI。現行做法應是：

- 年度：number input
- 月份：select 1～12 月
- 隱藏欄位 `#month` 保留 `YYYY-MM` 供程式使用。

### LINE Channel developing status

若未授權使用者按 `使用 LINE 登入` 後，在 LINE 頁面看到：

```text
400 Bad Request
This channel is now developing status. User need to have developer role.
```

原因不是排班系統程式錯誤，而是 LINE Developers 裡該 LINE Login Channel 還在 `Developing` 狀態。Developing 狀態只允許 Channel 的 Admin / Developer / Tester 等角色帳號登入，一般使用者會被 LINE 擋下。

處理方式：

1. 進入 LINE Developers Console。
2. 選擇正確 Provider 與 LINE Login Channel，目前 Channel ID：`2011263914`。
3. 到 Channel 的基本設定或頁面上方狀態區，將 Channel 從 `Developing` 發布為 `Published`。
4. 確認 Callback URL 仍是：`https://schedule.james-huang.org/auth/line/callback`。
5. 若暫時不想公開，只是找少數人測試，則把該 LINE 帳號加入 Channel 角色／測試人員；但正式申請流程應使用 `Published`。

發布後通常不需要重新部署 Docker；請未授權使用者重新開啟 `https://schedule.james-huang.org/login` 再按 LINE 登入。

### LINE Invalid redirect_uri

若 LINE 顯示：

```text
Invalid redirect_uri value. Check if it is registered in a LINE developers site.
```

處理：

1. 確認 Docker 的 `LINE_CHANNEL_ID` 是使用者在 LINE Developers 中設定 Callback URL 的那個 Channel。
2. Callback URL 必須完全一致：`https://schedule.james-huang.org/auth/line/callback`
3. 不能只登記首頁，也不要加尾端 `/`。
4. 如果 Callback URL 正確但仍錯，優先懷疑 Channel ID / Secret 不屬於同一個 LINE Login Channel。

### LINE 成功後又回登入頁

曾發生原因：`Set-Cookie` 覆蓋，導致 `yunlu_session` 沒送到瀏覽器。

修正原則：`setCookie()` 要讀取既有 `Set-Cookie` header，將新 cookie append 成陣列，而不是直接覆蓋。

### 核准申請時伺服器處理失敗

曾發生錯誤：

```text
TypeError: db.transaction is not a function
```

原因：Node 內建 `node:sqlite` 的 `DatabaseSync` 沒有 `db.transaction()`。

修正：使用手動 transaction：

```js
db.exec('BEGIN')
try {
  // writes
  db.exec('COMMIT')
} catch (err) {
  db.exec('ROLLBACK')
  throw err
}
```

## Procedure

### 修改流程

1. 先讀取相關檔案，不要憑記憶修改：
   - `C:\Users\HCH\Downloads\排班系統\index.html`
   - `C:\Users\HCH\Downloads\排班系統\serve.js`
   - 需要時讀 `C:\Users\HCH\Downloads\排班系統\開啟排班系統.bat`
2. 若改前端 UI 或前端排班／Excel 邏輯，修改 `index.html`。
3. 若改 API、SQLite、LINE Login、登入登出記錄、權限管理、port、刪除功能，修改 `serve.js`。
4. 若改啟動／重啟流程，修改 `開啟排班系統.bat`。
5. 修改後做語法檢查。
6. 若要同步公開版，依「公開版部署流程」scp + rebuild Docker。

## 驗證方式

### 前端語法檢查

```bash
python - <<'PY'
from pathlib import Path
p=Path(r'C:\Users\HCH\Downloads\排班系統\index.html')
s=p.read_text(encoding='utf-8')
js=s.split('<script>',1)[1].split('</script>',1)[0]
(p.parent/'_syntax.js').write_text(js,encoding='utf-8')
PY
node --check 'C:/Users/HCH/Downloads/排班系統/_syntax.js'
rm -f 'C:/Users/HCH/Downloads/排班系統/_syntax.js'
```

### 後端語法檢查

```bash
node --check 'C:/Users/HCH/Downloads/排班系統/serve.js'
```

### 本機 API 檢查

```bash
curl -s http://127.0.0.1:8766/api/schedules
curl -s http://127.0.0.1:8766/api/schedule?month=2026-09
```

### 公開版檢查

```bash
curl -fsS https://schedule.james-huang.org/health
ssh hch@10.145.119.12 "docker ps --filter name=yunlu-schedule --format '{{.Names}}|{{.Status}}|{{.Ports}}'"
ssh hch@10.145.119.12 "docker logs --tail 80 yunlu-schedule 2>&1"
```

登入登出記錄檢查：

```bash
# 需帶管理員登入 cookie 才會回資料；未登入應回 401，非管理員應回 403
curl -sS https://schedule.james-huang.org/api/login-events
```

未登入行為：

```bash
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' https://schedule.james-huang.org/
curl -sS -o /dev/null -w '%{http_code}\n' https://schedule.james-huang.org/api/schedules
```

預期：

- `/` 回 `302` 到 `/login`
- `/api/schedules` 回 `401`

### 使用者端提示

修改本機版完成後提醒使用者：

- 重新雙擊 `C:\Users\HCH\Downloads\排班系統\開啟排班系統.bat`
- 瀏覽器按 `Ctrl + F5`

修改公開版完成後提醒使用者：

- 開啟或重新整理 `https://schedule.james-huang.org`
- 若登入或權限狀態異常，先按 `登出`，再重新 LINE Login。

## 注意事項

- 不要覆蓋使用者的 `schedule.db`，除非使用者明確要求清除資料。
- 公開 Docker 重建時務必保留 `schedule.db` volume。
- 不要刪除 `template.xlsx`；它是保持 Excel 原格式的依據。
- 不要把 LINE Channel Secret 寫入 `index.html` 或回覆中。
- 不要把正式 Skill 放到 agent 私有目錄；本 Skill 正式位置是 `D:\OB\skills\yunlu-schedule-webapp\SKILL.md`。
- 若要大改資料結構，先考慮既有 SQLite JSON 與權限表是否需 migration。

## Conformance Addendum

## Inputs and Outputs
- **Input:** 使用者需求、目前本機與遠端部署狀態、相關路徑與錯誤訊息。
- **Output:** 修改後的本機／公開版排班系統、部署與驗證結果、必要時的還原或後續操作提示。

## Rules and Limitations
- 修改前必須讀取 `index.html` / `serve.js` 的現況，不要憑記憶改。
- 對 SQLite、Docker volume、LINE secret、Cloudflare Tunnel config 等有破壞風險的操作需保守處理。
- 機密不可外洩；LINE Channel Secret、SESSION_SECRET 不得出現在最終回覆。
- 現場檔案狀態優先於本 skill 的歷史紀錄。

## Verification
- 本機：`node --check serve.js` 與前端 script `node --check _syntax.js`。
- 公開版：`curl -fsS https://schedule.james-huang.org/health`。
- Docker：`docker ps --filter name=yunlu-schedule` 與 `docker logs --tail 80 yunlu-schedule`。
- 權限：未登入 `/api/schedules` 應回 401；未登入 `/` 應導向 `/login`。
