---
name: ilan-garbage-truck-query
description: >-
  Query Yilan County (宜蘭縣) garbage truck collection schedule and live vehicle position by
  address, using the public backend API behind https://clean.ilepb.gov.tw/garbageMap/ directly
  (no browser/Playwright needed), and optionally render the collection points + live trucks on an
  interactive Leaflet/Google-tile map (local HTML file, no Google Maps API key required). Use when
  the user asks "幫我查垃圾車" / "XX路幾號垃圾車幾點來" / "附近有垃圾車嗎" / "我要倒垃圾" /
  "用地圖顯示/標示清運點、車輛動態" for any Yilan address, or wants the nearest collection point,
  schedule, live truck GPS position, or a visual map near a given address in Yilan county.
---

# 宜蘭垃圾車查詢（直連背後 API，免開瀏覽器）

`https://clean.ilepb.gov.tw/garbageMap/` 這個官網其實只是個殼，真正的資料來自第三方廠商「群鴻科技
(EUP)」代管的通用查詢系統。這個 API **完全公開、CORS 全開（`access-control-allow-origin: *`）、無需
任何 API Key/Token**，可以直接用 curl/requests 呼叫，不必透過 Playwright 操作網頁。

## API 端點

```
POST https://customer-tw.eupfin.com/Eup_Servlet_Nuser_SOAP/Eup_Servlet_Nuser_SOAP
Content-Type: application/x-www-form-urlencoded;charset=UTF-8
Body: Param={...JSON...}   ← 注意：body 是 `Param=` 開頭接一個 JSON 字串，不是純 JSON body
```

回應固定格式：
```json
{"status": 1, "result": [...], "error": ""}
```
`status:1` = 成功；`status:3` = 查無資料/條件不符（正常情況，例如查得到路線但目前無提醒設定）。

## 標準查詢流程（地址 → 附近清運點+時刻表）

三支 API 依序呼叫，`MethodName` 決定要做什麼：

### 1. 地址轉座標
```json
Param={"Address":"宜蘭縣羅東鎮民權路206號","MethodName":"GetGisFromAddr"}
```
回應：`{"GISX":121.764587,"GISY":24.677097}`（度數浮點格式）

### 2. 座標查所屬清運責任區
```json
Param={"GISX":121764587,"GISY":24677097,"MethodName":"GetCustIDByLocation"}
```
⚠️ **注意座標格式在這裡要變成「整數 * 1e6」**（121764587，不是 121.764587）。
回應：`{"Cust_ID":"5034559"}`（各鄉鎮清運隊代碼）

### 3. 查附近清運點與時刻表
```json
Param={"Cust_ID":"5034559","Time":0,"RemovalType":0,"GISX":121.764587,"GISY":24.677097,"Range":300,"MethodName":"GetRecommandPoint"}
```
⚠️ 這裡座標格式**改回度數浮點**（跟步驟2相反，同一個值兩種寫法，呼叫時務必對應好）。
`Range` 是搜尋半徑（公尺，網站預設300，可調）。

回應為陣列，每筆清運點含：
- `PointName` — 點位名稱（如「民權路162號(奕順軒)【夜間定點】」）
- `Distance` — 距查詢座標的距離（公尺）
- `Details[].Time` — 表定抵達時刻
- `Details[].Week` — 收運星期（例：「每週一、二、四、五、日」）
- `RouteName` / `Route_ID` — 所屬路線
- `Car_Number` — 負責車號

**按 `Distance` 排序取最小值即為最近清運點。**

## 查詢附近垃圾車即時位置（車輛 GPS，非時刻表）

先用步驟2拿到的 `Cust_ID`，查該區域所有路線基本資料拿到 `Team_ID`：
```json
Param={"Cust_ID":"5034559","MethodName":"GetAllRouteBasicData"}
```
取任一筆的 `Team_ID`，再查即時車輛狀態：
```json
Param={"Cust_ID":5034559,"Team_ID":5033128,"MethodName":"GetCarStatusGarbage"}
```
回應為該清運隊「所有」垃圾車的最新回報位置（不是只回附近的），每筆含：
- `Car_Number` — 車牌
- `Log_GISX`/`Log_GISY` — 最新座標（度數浮點）
- `Log_DTime` — 最新回報時間
- `UseState` — `3`=出勤中，`0`=待命/未出勤
- `Log_Direct` — 行進方向角度

**要判斷「附近有沒有車」得自己用 Haversine/簡易平面距離公式算 `Log_GISX/GISY` 到查詢座標的距離**，
API 本身不會幫你篩選附近範圍（跟 `GetRecommandPoint` 不同，那支才有 `Range` 篩選）。

白天時段清運隊的車輛座標通常會叢集在完全不同的地點（例如車庫/上一輪收運的路線起點），只有真正接近
表定時刻（通常晚上）車輛才會出現在查詢地址附近，查詢前應先參考 `GetRecommandPoint` 拿到的表定時刻，
避免在離收運時間還很久時誤判「沒車」。

## 其他可用 MethodName（同一端點，換 Param 即可）

| MethodName | 用途 | 關鍵參數 |
|---|---|---|
| `GetRemovalWebSetting` | 網站設定（LOGO/電話/GoogleApiKey） | `TC_Code`（縣市代碼，宜蘭=19） |
| `GetCountryRemovalIsEnable` | 該縣市各鄉鎮清運隊清單（含`Team_ID`/`Cust_ID`/電話/座標） | `TC_Code` |
| `GetRoutePlanDetail` | 特定路線完整規劃明細 | `Route_ID` |
| `GetRecycleStation` | 全國資收站清單（不限宜蘭） | `DateTime` |
| `GetRemovalStatus` | 該清運隊車輛狀態（簡版，欄位少於 GetCarStatusGarbage） | `Cust_ID` |

## 在地圖上顯示清運點位+即時車輛（Leaflet.js 本機 HTML，免 Google Maps API Key）

這套工具正式落地在使用者的 **`D:\github\GarbageMapYilan\`**（使用者習慣把個人小工具集中放在 `D:\github` 下，一個工具一個資料夾），`skills/scripts/` 只是備份，**實際使用請直接去那邊執行**：

```
D:\github\GarbageMapYilan\
├─ RUN.cmd            ← 雙擊即可：互動輸入地址/範圍，自動查詢+產圖+用預設瀏覽器開啟
├─ scripts/fetch_map_data.js  ← 呼叫 API 產生 map_data.json
├─ scripts/gen_html_map.js    ← map_data.json -> garbage_map.html
└─ README.md
```

指令行用法（不想雙擊 RUN.cmd 時）：
```bash
cd /d/github/GarbageMapYilan
node scripts/fetch_map_data.js "宜蘭縣羅東鎮民權路206號" 300   # 地址 搜尋半徑(公尺)，產生 map_data.json
node scripts/gen_html_map.js                                     # map_data.json -> garbage_map.html
```
兩檔產出物（`map_data.json`/`garbage_map.html`）已列入 `.gitignore`。

若新環境沒有 `D:\github\GarbageMapYilan\` （例如在別台機器/容器內），才用 `scripts/` 備份，執行前先 `cd` 到
希望產出檔案的目錄（兩支腳本都用 `__dirname` 寫檔，不要直接在 skill 目錄裡寫出輸出檔）。

### 重要卡點：不能用官方網站的 Google Maps API Key

`GetRemovalWebSetting` 回應裏有一把 `GoogleApiKey`，看起來可以直接拿來嘅 Google Maps JavaScript API，但
**實測會被擋下來**（`RefererNotAllowedMapError`）——那把金鑰有網域白名單，只能跑在
`clean.ilepb.gov.tw` 底下，換到別的網址（包括本機 `file://` 或 `http://127.0.0.1`）都會被拒。

**解法**：改用 [Leaflet.js](https://leafletjs.com)（開源地圖庫，不需要 API Key）搭配 Google 的公開地圖圖砖端點：
```
https://mt{s}.google.com/vt/lyrs=m&x={x}&y={y}&z={z}   ← 道路圖
 https://mt{s}.google.com/vt/lyrs=s&x={x}&y={y}&z={z}   ← 衛星圖
```
（`{s}` 用 `0~3` 輪換）。不需要任何金鑰，直接當 tile URL 用。`scripts/gen_html_map.js` 已經內建了這套，
並帶道路/衛星切換按鈕。

### `file://` 協定無法直接用 Playwright 開嘅問題

Playwright MCP 這邊的 `browser_navigate` 會拒絕 `file://` 開頭的 URL（`Access to "file:" protocol is
blocked`）。若需要用 Playwright 截圖確認成果，要先起一個本機 HTTP server 把 HTML 目錄皆出來，例如：
```bash
node -e "
const http=require('http'),fs=require('fs'),path=require('path');
http.createServer((req,res)=>{
  const p=path.join(__dirname, req.url==='/'?'garbage_map.html':req.url);
  fs.readFile(p,(e,d)=>{ if(e){res.writeHead(404);res.end();return;} res.writeHead(200,{'Content-Type':'text/html'}); res.end(d); });
}).listen(8765);
" &
```
然後導航到 `http://127.0.0.1:8765/garbage_map.html`。

### 地圖上的互動內容

- 🔴 紅色圓點：查詢地址
- 🔵 藍色編號圓點：清運點位（點擊看路線名/表定時刻/星期/車號/距離）
- 🚛 橘色卡車圖示：即時垃圾車（`UseState:3`=出勤中，灰色=待命/未出勤），點擊看車號/狀態/最新回報時間
- 右上角：道路/衛星圖層切換
- 左下角「顯示所有車輛+點位」按鈕：白天時垃圾車常離查詢地址很遠（不同路線車庫/上一輪收運區），
  初始視野不會自動拉得足夠大能看到所有車，需要手動點這個按鈕才能 fitBounds 到全部點位。

## 實測範例（curl）

```bash
curl -s 'https://customer-tw.eupfin.com/Eup_Servlet_Nuser_SOAP/Eup_Servlet_Nuser_SOAP' \
  -H 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
  --data-urlencode 'Param={"Address":"宜蘭縣羅東鎮民權路206號","MethodName":"GetGisFromAddr"}'
```

## 這樣查 vs 開瀏覽器操作網站的取捨

- **純查詢時刻表/最近點位**：直接呼叫 API 最快，不需要 Playwright、不需要開 Chrome。
- **只需要單點地址的視覺化標示**（不需要清運點/車輛）：簡單用 Google Maps 靜態連結
  `https://www.google.com/maps?q=<lat>,<lng>&z=18`（見 `connect-chrome` skill 開分頁截圖）即可。
- **需要同時顯示多個清運點+即時車輛**：用上方「在地圖上顯示清運點位+即時車輛」的 Leaflet 本機 HTML 方案，
  一次看到所有點位與車輛、可點擊查詳情，這個宜蘭網站本身的地圖不支援插入自訂大頭針。

## 相關

- `connect-chrome` skill — 若需要用瀏覽器實際操作這個網站（例如要截圖給使用者看官方介面畫面）時使用。

---

## Conformance Addendum

## When to Use
Query Yilan County (宜蘭縣) garbage truck collection schedule and live vehicle position by address, using the public backend API behind https://clean.ilepb.gov.tw/garbageMap/ directly (no browser/Playwright needed), and optionally render the collection points + live trucks on an interactive Leaflet/Google-tile map (local HTML file, no Google Maps API key required). Use when the user asks "幫我查垃圾車" / "XX路幾號垃圾車幾點來" / "附近有垃圾車嗎" / "我要倒垃圾" / "用地圖顯示/標示清運點、車輛動態" for any Yilan address, or wants the nearest collection point, schedule, live truck GPS position, or a visual map near a given address in Yilan county.

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
