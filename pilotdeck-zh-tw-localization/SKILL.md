---
name: pilotdeck-zh-tw-localization
description: 當使用者要求把 Windows 上已安裝的 PilotDeck 介面改成台灣繁體中文、建立 zh-TW 語系、修復 PilotDeck 更新後的繁中補丁，或檢查 PilotDeck 是否支援繁中時使用。適用於 C:\\Program Files\\PilotDeck 的 Electron 安裝版；會備份 app.asar、停用中的 PilotDeck 進程、補上 zh-TW 選項、將既有 zh-CN 資源以 OpenCC s2twp 轉成台灣繁中，並驗證重啟後仍可使用。
---

# PilotDeck 台灣繁體中文本地化

## When to Use

使用者提到以下任一需求時載入：

- 「PilotDeck 改成繁體中文／台灣繁中」
- 「建立 PilotDeck 的 zh-TW 語系」
- 「PilotDeck 有 `zh-TW.pak`，但介面仍是英文或簡體」
- 「PilotDeck 更新後繁中補丁失效」
- 要檢查 `C:\\Program Files\\PilotDeck` 是否為 Electron 安裝版，或要替已安裝版本做本機 UI 本地化

本 Skill 針對 PilotDeck 桌面安裝版，不是 PilotDeck 上游原始碼的正式翻譯流程。原始版本可能只有 `en` 與 `zh-CN` 應用程式資源；`locales\\zh-TW.pak` 是 Chromium/Electron 的繁體語系檔，不代表 PilotDeck 自身 UI 已完成繁中翻譯。

## Inputs and Outputs

### Inputs

- PilotDeck 安裝根目錄，預設：`C:\\Program Files\\PilotDeck`
- PilotDeck 使用者資料目錄，通常：`C:\\Users\\<user>\\AppData\\Roaming\\pilotdeck-desktop`
- 可用的 Node.js/npm；若要做簡體到台灣繁中的自動轉換，需能取得 `opencc-js`（OpenCC `cn -> twp`）

### Outputs

- PilotDeck UI 語言選單新增 `繁體中文`，語言代碼為 `zh-TW`
- 既有簡體中文字串轉換為台灣繁體中文；未能安全轉換的字串保留原文或英文
- 啟動畫面、Electron 原生選單與主要 Web UI 使用繁中
- 安裝檔與 renderer 資源的可還原備份
- 驗證報告：安裝版本、修改檔案、備份位置、hash、重啟狀態

## Procedure

### 1. 查驗安裝與版本

先不要猜檔案名稱；確認目標是目前正在使用的安裝版本：

```powershell
$install = 'C:\Program Files\PilotDeck'
Test-Path "$install\PilotDeck.exe"
Test-Path "$install\resources\app.asar"
Test-Path "$install\resources\runtime\ui\dist\assets"
Get-Content "$install\resources\build-metadata.json" -Raw
Get-ChildItem "$install\locales" -Filter 'zh*.pak' | Select-Object Name,Length
```

正常會看到：

```text
locales\zh-CN.pak
locales\zh-TW.pak
resources\app.asar
resources\runtime\ui\dist\assets\index-<hash>.js
```

`zh-TW.pak` 只證明 Chromium locale 存在，不能直接讓 PilotDeck React UI 變成繁中。

### 2. 關閉 PilotDeck 並備份

這是會中斷桌面程式的操作，執行前要告知使用者；若使用者沒有授權關閉，先停在說明階段。

```powershell
Get-Process PilotDeck -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 2
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$backup = "D:\\OB\\backups\\pilotdeck-zh-tw-$stamp"
New-Item -ItemType Directory -Force $backup | Out-Null
Copy-Item "$install\resources\app.asar" "$backup\app.asar.original"
Copy-Item "$install\resources\runtime\ui\dist\assets\index-*.js" "$backup\" 
Copy-Item "$install\resources\runtime\ui\dist\assets\MemoryPanel-*.js" "$backup\" -ErrorAction SilentlyContinue
Copy-Item "$env:APPDATA\pilotdeck-desktop\appearance.json" "$backup\appearance.json.original" -ErrorAction SilentlyContinue
Get-ChildItem $backup | Select-Object Name,Length
```

若 PowerShell 在 Git Bash 內被呼叫，避免把 `$install` 等變數直接內嵌在 bash 字串；把 PowerShell 寫成 `.ps1` 再用 `powershell -File` 執行，以免被 bash 展開。

### 3. 讀取 Electron app.asar

使用 `@electron/asar` 檢查而不是直接以文字方式修改壓縮檔：

```powershell
npx --yes @electron/asar list "$install\resources\app.asar" | Select-String 'dist|appearance|applicationMenu|main.js'
npx --yes @electron/asar extract "$install\resources\app.asar" "$env:TEMP\\pilotdeck-app-asar"
```

重要檔案通常是：

```text
 dist/appearance.js
 dist/applicationMenu.js
 dist/main.js
```

### 4. 補上桌面 shell 的 zh-TW 支援

在解開的 `dist/appearance.js` 中：

1. 讓 `normalizeAppearance()` 接受 `zh-TW`，不要只接受 `en` 與 `zh-CN`。
2. 新增 `STARTUP_ZH_TW`，至少翻譯啟動、停止、錯誤、重試、開啟紀錄等訊息。
3. `startupText()` 在 `language === 'zh-TW'` 時使用 `STARTUP_ZH_TW`。
4. loading HTML 的 translations 也要在 `zh-TW` 時使用繁中表。

在解開的 `dist/applicationMenu.js` 中：

1. 將語言判斷改成 `language.startsWith('zh')`，使 `zh-CN` 與 `zh-TW` 都能顯示中文選單。
2. 對 `zh-TW` 使用台灣用字，例如：
   - `文件` → `檔案`
   - `编辑` → `編輯`
   - `撤销` → `復原`
   - `复制` → `複製`
   - `粘贴` → `貼上`
   - `查看` → `檢視`
   - `窗口` → `視窗`
   - `重新加载界面` → `重新載入介面`
   - `切换全屏` → `切換全螢幕`

修改後先驗證：

```powershell
node --check "$env:TEMP\\pilotdeck-app-asar\\dist\\appearance.js"
node --check "$env:TEMP\\pilotdeck-app-asar\\dist\\applicationMenu.js"
```

### 5. 補上 Web UI 的 zh-TW 選項與資源

PilotDeck 的主要 UI 在安裝目錄外的 runtime bundle，不在 app.asar 內。先找出 hash 資產：

```powershell
$asset = Get-ChildItem "$install\resources\runtime\ui\dist\assets" -Filter 'index-*.js' | Select-Object -First 1
$memory = Get-ChildItem "$install\resources\runtime\ui\dist\assets" -Filter 'MemoryPanel-*.js' | Select-Object -First 1
$asset.FullName
$memory.FullName
```

bundle 內原本可觀察到：

```javascript
{value:"en", ...}, {value:"zh-CN", ...}
resources: { en: {...}, "zh-CN": {...} }
```

需要做以下修改：

1. 在 onboarding 語言清單加入：

```javascript
{value:"zh-TW", nameKey:"language.tw.name", hintKey:"language.tw.hint"}
```

2. 在設定頁語言清單加入：

```javascript
{value:"zh-TW", label:"Traditional Chinese", nativeName:"繁體中文"}
```

3. 將 onboarding 的目前選取判斷改成先辨識 `zh-TW`，不能把所有 `zh*` 都強制視為 `zh-CN`。
4. 建立 `zh-TW` namespace：以既有 `zh-CN` resource bundle 為基礎轉換到台灣繁體。
5. 在 onboarding namespace 補上：

```text
language.tw.name = 繁體中文
language.tw.hint = Chinese · Traditional (Taiwan)
```

6. 語言變更後要讓桌面 shell 收到 `zh-TW`，而不是把它壓回 `zh-CN`。
7. `MemoryPanel-*.js` 的語言判斷也應使用 `language.startsWith("zh")`，否則 Memory 面板在 `zh-TW` 會退回英文。

### 6. 由簡體轉成台灣繁中

不要只做逐字替換。使用 OpenCC 的 `cn -> twp`，因為 `twp` 會同時處理字形與台灣常用詞：

```javascript
const OpenCC = require('opencc-js');
const convert = OpenCC.Converter({from: 'cn', to: 'twp'});
convert('简体中文 软件 网络 数据库 文件夹');
// 簡體中文 軟體 網路 資料庫 資料夾
```

若直接以 OpenCC package API 產生轉換表，需注意 Windows Node 的路徑與 ESM：

```text
使用 file:///C:/... 的 file URL 載入 ESM，不能直接 import 'C:/...'
```

安全做法是：

- 遞迴處理 resource object 的 string value、array 與 nested object。
- 優先保留 key、placeholder、HTML tag、Markdown 語法及 model ID。
- 只轉換實際顯示文字。
- 對產品名 `PilotDeck`、`Gateway`、API 名稱及路徑不要轉換。
- 轉換完成後抽查「軟體、網路、資料、檔案、設定、執行、檢視、視窗」等台灣用字。

此方法是本機衍生翻譯，不是 PilotDeck 官方繁體中文翻譯；更新後若上游改了 zh-CN 資源，應重新產生而不是沿用舊 bundle。

### 7. 重建 app.asar 與安裝檔資源

把修改後的 extracted app 重新打包：

```powershell
npx --yes @electron/asar pack "$env:TEMP\\pilotdeck-app-asar" "$env:TEMP\\pilotdeck-app-patched.asar"
```

將新的 app.asar 與外部 renderer bundle 複製回 `Program Files` 前，必須使用系統管理員權限；不要刪除原檔：

```powershell
Start-Process powershell.exe -Verb RunAs -Wait -ArgumentList @(
  '-NoProfile', '-ExecutionPolicy', 'Bypass', '-Command',
  "Copy-Item -LiteralPath '$env:TEMP\\pilotdeck-app-patched.asar' -Destination '$install\\resources\\app.asar' -Force; Copy-Item -LiteralPath '$env:TEMP\\pilotdeck-index-patched.js' -Destination '$asset' -Force; Copy-Item -LiteralPath '$env:TEMP\\pilotdeck-memory-patched.js' -Destination '$memory' -Force"
)
```

如果系統管理員複製仍回報 Access Denied，先確認所有 PilotDeck、renderer、runtime node 子進程都已結束，再重試；不要用 `takeown` 或修改整個 Program Files 權限作為第一選項。

### 8. 設定使用者偏好並重啟

```powershell
$appearance = Join-Path $env:APPDATA 'pilotdeck-desktop\appearance.json'
Set-Content -LiteralPath $appearance -Value '{"language":"zh-TW","themeMode":"system"}' -Encoding utf8
Start-Process "$install\PilotDeck.exe"
```

Chromium localStorage 的 `userLanguage` 可能仍保存 `en` 或 `zh-CN`。若啟動後沒有自動採用 zh-TW，應在 PilotDeck 設定頁切換一次「繁體中文」，讓 React/i18next 寫入 `userLanguage = zh-TW`；不要直接修改 LevelDB，因為資料庫有 lock 與編碼格式。

### 9. 更新後重新套用

PilotDeck 自動更新會覆蓋：

```text
resources/app.asar
resources/runtime/ui/dist/assets/index-<hash>.js
resources/runtime/ui/dist/assets/MemoryPanel-<hash>.js
```

每次更新後：

1. 讀取 `resources/build-metadata.json` 的 version、commitSha。
2. 重新查找新的 hash bundle，不要假設舊檔名仍存在。
3. 以新版本的 zh-CN resource 為來源重新產生 zh-TW。
4. 先備份，再重新打包與複製。
5. 重啟並再次驗證。

## Pitfalls

- `locales\\zh-TW.pak` 是 Electron/Chromium 語系檔，不是 PilotDeck React UI 的翻譯資源。
- 原版 PilotDeck 的應用程式 locale 可能只有 `en` 與 `zh-CN`；不能只把 `zh-CN` 字串搜尋取代成 `zh-TW`，必須新增 resource bundle 與語言選項。
- hashed asset 檔名會隨版本改變；先用 `Get-ChildItem` 或 `find` 找檔案，再修改。
- `app.asar` 與 `runtime/ui` 是兩個不同位置；只修改其中一邊會造成啟動畫面是繁中、主 UI 仍是英文，或反過來。
- `Program Files` 需要系統管理員權限；Access Denied 不代表補丁內容錯誤。
- 修改前一定要關閉所有 PilotDeck 子進程，否則 app.asar 或 bundle 可能被鎖定。
- 不要直接編輯 Chromium localStorage LevelDB；應透過 UI 切換語言，或讓 bundle 的初始語言邏輯使用 `zh-TW`。
- OpenCC `cn -> t` 只做標準繁體，不一定符合台灣詞彙；面向使用者的 UI 應優先使用 `cn -> twp`。
- 不要把產品名稱、API provider、model ID、檔案路徑或 placeholder 轉換成中文。
- 這是本機修改版，不是官方 PilotDeck 繁中翻譯；自動更新、重新安裝或檔案完整性修復可能覆蓋修改。
- 不要為了測試而刪除使用者資料目錄、Cookie、設定、runtime log 或 model config。

## Verification

### 檔案與語法

```powershell
Test-Path "$install\resources\app.asar"
Test-Path "$install\resources\runtime\ui\dist\assets\index-*.js"
node --check "$env:TEMP\\pilotdeck-app-asar\\dist\\appearance.js"
node --check "$env:TEMP\\pilotdeck-app-asar\\dist\\applicationMenu.js"
```

### 補丁內容

```powershell
Select-String -Path "$install\resources\runtime\ui\dist\assets\index-*.js" -Pattern 'zh-TW','繁體中文','zhTWChars'
Select-String -Path "$install\resources\runtime\ui\dist\assets\MemoryPanel-*.js" -Pattern 'startsWith\("zh"\)'
npx --yes @electron/asar list "$install\resources\app.asar" | Select-String 'dist/appearance.js|dist/applicationMenu.js'
```

應能看到 `zh-TW`、`繁體中文`，且 app.asar 包含 shell 檔案。

### 執行狀態

```powershell
Get-Process PilotDeck -ErrorAction SilentlyContinue | Select-Object Id,Path,StartTime
Get-Content "$env:APPDATA\pilotdeck-desktop\appearance.json" -Raw
```

應看到 PilotDeck 正常啟動，appearance 設定為：

```json
{"language":"zh-TW","themeMode":"system"}
```

### UI 人工檢查

至少確認：

1. 設定頁語言下拉選單顯示「繁體中文」。
2. 切換後重新整理或重啟，語言仍為繁中。
3. 原生選單顯示「檔案、編輯、檢視、視窗」。
4. 啟動畫面、重試、開啟紀錄等文字使用繁中。
5. 主要頁面可看到「設定、檔案、工作區、對話、模型」等繁體字。
6. Memory 面板不因 `zh-TW` 而退回英文。
7. PilotDeck runtime server 與 Gateway 都能正常啟動。

### 還原

若啟動失敗，先關閉 PilotDeck，再以備份還原：

```powershell
Copy-Item "$backup\app.asar.original" "$install\resources\app.asar" -Force
Copy-Item "$backup\index-*.js" "$install\resources\runtime\ui\dist\assets\" -Force
Copy-Item "$backup\MemoryPanel-*.js" "$install\resources\runtime\ui\dist\assets\" -Force -ErrorAction SilentlyContinue
```

還原後重新啟動，並保留錯誤訊息與 runtime log 供後續分析。

## Conformance Addendum

## Inputs and Outputs
- **Input:** PilotDeck 安裝路徑、目前版本與使用者是否授權關閉/重啟應用程式。
- **Output:** 可還原的 PilotDeck zh-TW 本地化修改、備份路徑與驗證結果。

## Rules and Limitations
- 修改 `Program Files` 前必須先備份；不要在未授權時關閉使用者正在使用的 PilotDeck。
- 不處理帳號密碼、API key、Cookie 或 model credentials。
- 只修改語系與顯示資源，不修改 runtime 的模型、網路、權限或資料庫設定。
- 每次更新後以當前安裝檔案為準，不假設舊版 hash、行號或 bundle 結構仍相同。
