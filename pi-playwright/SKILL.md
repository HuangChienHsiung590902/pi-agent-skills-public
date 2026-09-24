---
name: pi-playwright
description: >-
  在這台 Windows 電腦安裝、更新、檢查與使用一般 Playwright、Playwright CLI、@playwright/test，或在 pi coding agent 裡執行網頁自動化。只要使用者提到 Playwright 安裝、playwright CLI、npx playwright、codegen、test、screenshot、瀏覽器 binary、缺少 playwright 套件，或要求用全新測試瀏覽器開頁、點擊、填表、截圖、檢查 console/network，就使用本 Skill。若要接管已開啟且保留登入狀態的 Chrome，改用 connect-chrome／pi-playwright-existing-chrome；若要 CloakBrowser 隱身 Chromium，改用 cloakbrowser-automation；瀏覽器原生 UI 或桌面程式不適合用 Playwright，改用能控制 Windows 桌面 UI 的專用工具。
compatibility: Windows、Node.js、npm；一般 CLI 需要全域或專案安裝 playwright／@playwright/test，pi 整合則需要 pi-playwright 套件。
---

# Playwright 與 pi-playwright

## When to Use

使用於：

- 檢查這台電腦有沒有完整 `playwright`、`@playwright/test`、`playwright-core`、CLI 與瀏覽器 binary。
- 安裝、更新或修復一般 Playwright CLI／Playwright Test。
- 執行 `playwright open`、`codegen`、`screenshot`、`pdf`、`test`、`show-report`、`show-trace`。
- 在 pi coding agent 裡使用 `pi-playwright` 做開頁、DOM snapshot、點擊、填表、截圖、console/network 檢查與登入 state 存取。
- 排查「套件已裝但 CLI 找不到」、「只有 playwright-core」、「瀏覽器 executable 不存在」等問題。

不要用於：

- 接管使用者已開啟、已有登入狀態的 Chrome：使用 `connect-chrome` 或 `pi-playwright-existing-chrome`。
- CloakBrowser／反偵測 Chromium：使用 `cloakbrowser-automation`。
- 瀏覽器工具列、原生檔案選擇器或桌面 App：Playwright 不適用，改用能控制 Windows 桌面 UI 的專用工具。
- 單純因 CloakBrowser 需要 `playwright-core` 而安裝完整 Playwright；CloakBrowser 可獨立運作。

## Inputs and Outputs

### Inputs

- 需求屬於一般 CLI、Playwright Test、Node.js API，還是 pi 的 `pi-playwright` wrapper。
- 目標 URL、測試目錄、輸出路徑，以及是否 headed／headless。
- 是否需要新的隔離瀏覽器，或其實要接管既有 Chrome。

### Outputs

- 安裝／版本／binary 狀態與實際驗證結果。
- 使用者要求的截圖、PDF、trace、report 或測試結果。
- 若有失敗，回報實際命令、錯誤與下一步，不把 `playwright-core` 誤報成完整 Playwright。

## Local Baseline

目前已驗證基準（每次仍以實際檢查為準）：

- Node.js：`v24.21.0`
- 全域 `playwright`：`1.63.0`
- 全域 `@playwright/test`：`1.63.0`
- 全域 `playwright-core`：`1.63.0`
- CLI：`playwright --version` 可用
- Playwright browser cache：`C:\Users\Administrator\AppData\Local\ms-playwright`
- Chromium revision：`1243`（Chrome for Testing `153.0.8010.12`）
- pi 套件預期位置：`~/.pi/agent/npm/node_modules/pi-playwright/`

版本與路徑可能在更新後改變；不要只憑本段宣稱現況。

## Procedure

### 1. 判斷是哪一種 Playwright

先區分：

1. **一般 Playwright CLI**：快速開頁、codegen、截圖、PDF、trace。
2. **Playwright Test**：fixtures、assertions、平行測試、retry、HTML report。
3. **Playwright library API**：Node.js 腳本中的 `chromium.launch()` 等 API。
4. **pi-playwright**：pi 專用 wrapper，底層呼叫本機 `@playwright/cli`／`playwright-cli`。

Playwright CLI 是 Playwright 的命令列入口，不是可脫離 Playwright 套件的替代產品。

### 2. 先檢查，不要直接重裝

```powershell
playwright --version
npx --no-install playwright --version
npm list -g --depth=0 playwright @playwright/test playwright-core
npm list --depth=0 playwright @playwright/test playwright-core
playwright install --list
```

判讀：

- 只有 `playwright-core`：底層 API 存在，但通常沒有完整 CLI、Test runner 與 browser download 管理體驗。
- `npx --no-install` 失敗：目前解析範圍內沒有可用的完整 `playwright` 套件；它不會偷偷下載最新版，因此適合檢查。
- 全域安裝不代表專案具備可重現依賴；正式專案仍應本地安裝並提交 lockfile。

### 3. 依用途安裝

#### 本機通用 CLI

```powershell
npm install -g playwright@latest @playwright/test@latest
playwright install chromium
```

只有使用者需要 Firefox／WebKit 時才安裝：

```powershell
playwright install firefox webkit
```

#### 專案測試（建議）

在專案根目錄：

```powershell
npm install -D @playwright/test
npx playwright install chromium
```

`@playwright/test` 會帶入匹配版本的 Playwright；除非程式碼明確直接依賴 `playwright` library，否則不必重複加入兩套頂層依賴。

#### 只有 library API

```powershell
npm install playwright
```

### 4. 驗證 CLI 與瀏覽器真的能啟動

```powershell
playwright --version
playwright install --list
playwright screenshot --browser chromium https://example.com C:\Users\Administrator\playwright-smoke-test.png
```

確認：

- 命令 exit code 為 0。
- screenshot 檔案存在且非空。
- `install --list` 列出 Chromium 與其路徑。

不要只用 `npm list` 當成功證據；套件存在不保證 browser binary 能啟動。

### 5. 使用一般 Playwright CLI

```powershell
playwright open https://example.com
playwright codegen https://example.com
playwright screenshot https://example.com screenshot.png
playwright pdf https://example.com page.pdf
playwright test
playwright show-report
playwright show-trace trace.zip
```

先用 `playwright --help` 與 `playwright <command> --help` 確認目前版本支援的參數。

### 6. 在 pi 裡使用 pi-playwright

`pi-playwright` 是另一層整合，不等同剛安裝的全域 `playwright`。先找出實際 skill 目錄，不要硬套舊使用者名稱：

```powershell
$SkillDir = Join-Path $HOME ".pi\agent\npm\node_modules\pi-playwright\skills\playwright-browser"
Test-Path $SkillDir
```

常用操作：

```powershell
node "$SkillDir\scripts\pw.js" open https://example.com
node "$SkillDir\scripts\pw.js" snapshot --filename "C:\path\snapshot.md"
node "$SkillDir\scripts\pw.js" fill e4 "alice@example.com"
node "$SkillDir\scripts\pw.js" click e5
node "$SkillDir\scripts\pw.js" eval "() => document.title"
node "$SkillDir\scripts\pw.js" console
node "$SkillDir\scripts\pw.js" network
node "$SkillDir\scripts\pw.js" screenshot --filename "C:\path\page.png" --full-page
node "$SkillDir\scripts\pw.js" state-save "C:\path\auth.json"
node "$SkillDir\scripts\pw.js" close
```

需要完整參數時，讀實際套件內的 `references/commands.md`。

## Windows Troubleshooting

### CLI 找不到，但 npm 顯示已安裝

```powershell
npm prefix -g
npm root -g
Get-Command playwright -ErrorAction SilentlyContinue
```

確認 npm global bin 所在目錄在目前使用者的 `PATH`。安裝後舊終端若仍找不到，重開終端再測。

### pi-playwright 指令 exit code 1 且沒有輸出

部分版本在 Windows 可能遇到：

- wrapper 假設 `playwright-cli.cmd` 位於套件自己的 `node_modules\.bin`，但 npm hoist 到上層。
- Node.js `spawnSync` 直接執行 `.cmd` 時未經 shell。

先檢查實際版本與檔案，不要盲目修改：

```powershell
Get-ChildItem "$HOME\.pi\agent\npm\node_modules" -Filter playwright-cli.cmd -Recurse -ErrorAction SilentlyContinue
```

若可確認是上述問題，備份 `skills/playwright-browser/scripts/lib/runtime.js` 後：

- 讓 CLI resolver 從 package root 往父層尋找 `node_modules/.bin/playwright-cli.cmd`。
- Windows 執行 `.cmd` 時設定 `shell: process.platform === "win32"`。

重新安裝／更新 `pi-playwright` 可能覆蓋本機修補，更新後需重新驗證。

### Playwright 與 CloakBrowser binary 不共用

一般 Playwright 的 browser cache 通常位於：

```text
%LOCALAPPDATA%\ms-playwright
```

CloakBrowser binary 通常位於：

```text
%USERPROFILE%\.cloakbrowser
```

兩者用途、版本與啟動方式不同；不要把 CloakBrowser 已安裝誤當成 Playwright Chromium 已安裝，反之亦然。

## Rules and Limitations

- 只對使用者擁有、管理或獲准的網站執行自動化。
- 不協助繞過 CAPTCHA、登入／權限控制、付費牆、封鎖、速率限制或其他安全機制。
- 發文、付款、送出表單、刪除或帳號變更等外部副作用，執行前確認目標與內容。
- 不把帳密、cookies、token 或 proxy 密碼寫入腳本、log、skill 或回覆；使用環境變數或由使用者自行登入。
- 專案依賴優先本地安裝；全域安裝適合本機通用 CLI，但不能取代專案 lockfile。
- 不為了「補齊」而安裝所有瀏覽器；預設只裝 Chromium，除非需求明確需要 Firefox／WebKit。
- 相對輸出路徑可能受目前工作目錄影響；重要產出使用絕對路徑。

## Pitfalls

- `playwright-core` 不等於完整 `playwright`，也不等於 `@playwright/test`。
- `npx playwright` 在套件缺少時可能詢問或下載最新版；檢查安裝狀態用 `npx --no-install playwright --version`。
- 全域 CLI 可用不代表 `import { chromium } from "playwright"` 在任意專案都可解析；Node.js 專案應安裝本地依賴。
- `playwright install chromium` 只安裝 Playwright 管理的 Chromium，不會安裝或更新 CloakBrowser。
- 新開的 Playwright browser 不會自動繼承一般 Chrome 登入狀態。
- `codegen`、HTML report 與 trace viewer 通常需要 headed 視窗，不適合純 headless 環境。
- 不要把記錄的版本與路徑視為永久；先查當前狀態。

## Verification

### 一般 Playwright

1. `playwright --version` 成功並輸出版本。
2. `npm list -g --depth=0 playwright @playwright/test playwright-core` 列出預期套件。
3. `playwright install --list` 列出 Chromium binary。
4. 對 `https://example.com` 執行 screenshot smoke test，輸出檔存在且非空。
5. 若是專案安裝，再執行 `npx playwright test` 或最小 smoke test。

### pi-playwright

1. 實際 `pi-playwright` skill 目錄存在。
2. `pw.js open https://example.com` 成功。
3. snapshot 能取得頁面文字／DOM refs。
4. screenshot 檔案存在。
5. 執行 `close` 後沒有遺留無主 Chromium process。

回報時分別列出套件版本、browser 安裝位置、實際 smoke test，以及尚未測試的部分。
