---
name: obs-windows-black-video-diagnose
description: 診斷與修復 Windows OBS「來源明明啟用、直播正常，但預覽／錄影沒有影像或全黑」問題，尤其是瀏覽器來源、YouTube 直播來源被誤移到畫布外、縮放或裁切異常、來源未顯示、OBS 設定檔與即時狀態不一致。當使用者提到 OBS 沒影像、黑畫面、來源消失、YouTube 直播錄不到、預覽空白、來源座標跑掉，或要透過 SSH 遠端檢查 OBS 時，務必使用此 Skill；優先用 obs-websocket 讀取與修復即時狀態，不要在 OBS 運作中直接改 scene JSON。
---

# Windows OBS 黑畫面診斷與修復

## 核心經驗

OBS 的來源可同時處於「已啟用、可見、尺寸正常」，但因 `positionX`／`positionY` 極端異常而完全落在畫布外，造成預覽與錄影看起來沒有影像。

已確認案例：1920×1080 的 YouTube 瀏覽器來源被設為 `X=-2209592, Y=0`；YouTube 頁面在線、OBS 正常回應、來源也為 visible，但畫面完全在左側畫布外。透過 obs-websocket 把即時 transform 改回 `X=0, Y=0, scale=1` 後立即恢復，再鎖定來源避免誤拖。

**最重要的判斷原則：**

1. SSH 登入帳號不一定是執行 OBS 的桌面帳號。先找出 `obs64.exe` 所在 session、程序擁有者與實際 `%APPDATA%`；不要預設使用 SSH 帳號的 profile。
2. scene collection JSON 只適合初步診斷與離線備份。OBS 執行中，磁碟 JSON 可能尚未寫回，甚至持續顯示舊座標。
3. 修復與最終驗證均以 **obs-websocket 即時狀態**為準。
4. 不要因 YouTube 的廣告 CORS、sandbox、`about:blank` console 訊息就直接判定影片失效；這些常是非致命噪音。

## Inputs and Outputs

### Inputs

- 本機或已獲授權的遠端 Windows 主機。
- OBS 執行狀態、場景、來源、日誌與 canvas/output 解析度。
- 若要自動修復：已啟用的 obs-websocket 連接埠與密碼，經環境變數或互動輸入提供。

### Outputs

- 根因分類與證據：程序、來源可見性、鎖定狀態、即時座標、尺寸、URL、日誌、磁碟與 GPU 狀態。
- 經使用者授權後完成的最小修復，例如把來源移回 `(0,0)`、恢復縮放、啟用來源及鎖定。
- 修復後由 obs-websocket 讀回的即時驗證結果。

## 安全規則

- 僅操作使用者明確授權的主機與 OBS instance。
- 不把 SSH、OBS WebSocket 密碼、stream key、cookie 或 token 寫入 Skill、腳本、日誌或回覆。
- 密碼以環境變數（如 `OBS_WS_PASSWORD`）或安全提示傳入；命令輸出要遮蔽秘密。
- 修復前先查詢；涉及停止錄影、關閉 OBS、重啟服務、覆寫場景檔或刪除 cache，必須先取得使用者確認。
- 修改前記錄原始 transform；只做能解決問題的最小變更。

## 診斷流程

### 1. 確認 OBS 與實際桌面使用者

在遠端 Windows PowerShell 檢查：

```powershell
Get-CimInstance Win32_Process |
  Where-Object Name -in @('obs64.exe','obs-browser-page.exe') |
  Select-Object Name,ProcessId,SessionId,CreationDate,ExecutablePath,CommandLine
quser
Get-Process obs64 -ErrorAction SilentlyContinue |
  Select-Object Id,SessionId,StartTime,Responding,CPU,WorkingSet
```

若 SSH 使用者與 OBS 桌面使用者不同，後續 OBS 資料路徑應指向後者，例如：

```text
C:\Users\<OBS桌面使用者>\AppData\Roaming\obs-studio
```

### 2. 排除基礎環境問題

檢查：

- OBS 是否 `Responding=True`。
- 系統碟與錄影碟是否還有空間。
- GPU 裝置／驅動是否正常。
- 最近是否仍產生錄影檔，最後修改時間與檔案大小是否合理。
- 最新 OBS log 是否有 encoder、render、device、recording stop、crash 等實質錯誤。

`aja.dll`、DeckLink、VLC 未安裝、NVENC DLL 不存在等訊息，若功能本來未使用，通常不是瀏覽器來源黑畫面的根因。

### 3. 檢查場景與來源設定

場景檔通常位於：

```text
%APPDATA%\obs-studio\basic\scenes\*.json
```

初步檢查：

- `current_program_scene` 與 `current_scene`
- 來源 `id`（例如 `browser_source`）
- `settings.url`、`width`、`height`
- scene item 的 `visible`、`locked`
- `pos.x`、`pos.y`、`scale.x`、`scale.y`
- crop 與 bounds

判斷來源是否在畫布外：

```text
itemRight  = positionX + width  * abs(scaleX)
itemBottom = positionY + height * abs(scaleY)
```

若 `itemRight <= 0`、`itemBottom <= 0`、`positionX >= canvasWidth` 或 `positionY >= canvasHeight`，來源完全位於畫布外。極大絕對座標（例如數萬至數百萬像素）通常是誤拖或自動化設定錯誤。

**注意：** JSON 是診斷線索，不是 OBS 運作中最終真相。若 JSON 與畫面不符，改用下一步查即時狀態。

### 4. 用 obs-websocket 查詢即時狀態

OBS 5.x 預設 WebSocket 常為 `4455`。先檢查：

```powershell
Get-NetTCPConnection -State Listen |
  Where-Object LocalPort -in 4455,4444
```

使用 bundled script：

```powershell
$env:OBS_WS_PASSWORD = '<由使用者安全提供，不要寫進檔案>'
& .\scripts\obs-scene-item.ps1 -Port 4455 -List
```

指定場景與 item ID 查詢：

```powershell
& .\scripts\obs-scene-item.ps1 -Port 4455 -SceneName '場景' -SceneItemId 1
```

回報至少包含：

- `sceneItemEnabled`
- `sceneItemLocked`
- `positionX`, `positionY`
- `width`, `height`
- `sourceWidth`, `sourceHeight`
- `scaleX`, `scaleY`

### 5. 修復畫布外來源

先向使用者說明將修改哪個來源；取得授權後執行：

```powershell
& .\scripts\obs-scene-item.ps1 `
  -Port 4455 `
  -SceneName '場景' `
  -SceneItemId 1 `
  -RepairToOrigin `
  -Enable `
  -Lock
```

此操作透過 WebSocket：

1. 讀取並輸出修改前狀態。
2. 設定 `positionX=0`、`positionY=0`。
3. 預設保留原縮放；若使用 `-ResetScale` 才設為 `scaleX=1, scaleY=1`。
4. 視參數啟用並鎖定來源。
5. 再次讀取即時狀態驗證。

若希望自動鋪滿畫布，不要猜測比例；可在 OBS UI 使用「轉換 → 符合螢幕大小」，或明確取得 canvas 與來源尺寸後計算等比例縮放。

### 6. 驗證

修復完成必須確認：

- 即時 `positionX/positionY` 已回到合理範圍。
- `width/height` 非 0。
- `sceneItemEnabled=True`。
- 若使用者已按鎖頭，`sceneItemLocked=True`。
- OBS 仍正常回應。
- 使用者看到影像，或新錄製的短測試片段確實含有畫面。

不要只讀 scene JSON 就宣稱成功；OBS 可能尚未把記憶體狀態寫回檔案。`SaveSceneCollection` 並非所有 obs-websocket 版本都支援，收到 unsupported request（常見 code 204）時不要反覆重試。

## YouTube 瀏覽器來源的額外判斷

若來源 URL 是頻道 `/live`：

1. 以一般 HTTP 請求確認回應碼與頁面是否包含影片資訊，只作可用性線索。
2. 檢查是否顯示 offline、登入／年齡限制、地區限制、同意頁或 DRM 問題。
3. CORS 廣告請求失敗與 sandboxed `about:blank` 常是非致命訊息；必須與即時 transform、來源尺寸及實際畫面一起判斷。
4. 若 transform 正常仍全黑，可在 OBS UI 對瀏覽器來源執行一次「重新整理目前頁面」，但不要形成快速 reload 重試風暴。

## 常見陷阱

- **找錯 profile：** SSH 是 `Administrator`，OBS 卻由 `James` 執行，導致誤以為 logs 不存在。
- **只看 visible：** `visible=True` 不代表來源位於畫布內。
- **直接編輯 JSON：** OBS 執行中可能覆蓋你的修改，且改壞 JSON 會損害整個 scene collection。
- **磁碟狀態落後：** WebSocket 已是 `(0,0)`，JSON 仍保留舊座標；以即時 API 為準。
- **把無關 plugin 警告當根因：** 未使用的 AJA、DeckLink、NVENC、VLC plugin 警告不一定影響 browser source。
- **未鎖來源：** 修好後未鎖定，使用者可能再次誤拖。
- **洩漏憑證：** 不可列印 obs-websocket config 中的明文密碼；只讀 port、enabled、auth_required 等非秘密欄位。

## 回報格式

```text
診斷結果：<根因>
證據：<來源、可見狀態、即時座標、尺寸與必要日誌>
已處理：<實際做過的最小修改>
驗證：<WebSocket 即時讀回值與使用者畫面確認>
仍需注意：<尚未驗證或可能復發條件>
```
