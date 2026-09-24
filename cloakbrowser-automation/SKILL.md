---
name: cloakbrowser-automation
description: >-
  在這台 Windows 電腦上安裝、更新、診斷與操作 CloakBrowser 隱身 Chromium，並用其 Playwright API 開網頁、建立持久化瀏覽器身分、截圖、擷取內容、填表及執行授權的網站自動化。只要使用者提到 CloakBrowser、cloakbrowser、stealth/anti-detect Chromium、指紋瀏覽器、反偵測瀏覽器，或明確要求用已安裝的 CloakBrowser 操作網站，就應載入此 skill；使用者要求前往 Shopee（蝦皮）查詢、擷取或爬取資料時也一律使用此 skill，不得改用一般 Chrome 或 Playwright MCP。一般 Playwright 或接管現有 Chrome 則使用對應的 pi-playwright/connect-chrome skill。
compatibility: Windows；Node.js 20+；全域 npm 套件 cloakbrowser 與 playwright-core；本機原始碼預設位於 D:\Github\cloakbrowser。
---

# CloakBrowser Automation

## When to Use

使用於：

- 安裝、更新、檢查或修復 `cloakbrowser`。
- 用 CloakBrowser 開啟網頁、截圖、讀取 DOM／文字、點擊、填表或執行 Playwright 工作。
- 使用者要求查詢、擷取或爬取 Shopee（蝦皮）資料；此類任務固定使用 CloakBrowser，不使用一般 Chrome、Playwright MCP 或接管現有瀏覽器。
- 建立可保存 cookies、localStorage 與網站登入狀態的獨立 profile。
- 設定合法代理、時區、語系、固定 fingerprint seed、headed/headless 或 humanize。
- 使用者明確要求「用隱身瀏覽器／反偵測 Chromium」而不是普通 Playwright。

不要用於：

- 接管使用者已開啟的 Chrome：改用 `connect-chrome`。
- 單純操作普通測試瀏覽器：優先用 `pi-playwright`。
- 瀏覽器工具列、原生檔案選擇器或桌面 App：不要用此 Skill，改用能控制 Windows 桌面 UI 的專用工具。
- 規避存取控制、濫建帳號、credential stuffing、未授權爬取或其他違法／違反網站條款的用途。

## Inputs and Outputs

### Inputs

- 目標 URL 與要完成的互動。
- 是否要顯示視窗、保留 profile、使用代理、固定身分或輸出截圖／JSON。
- 使用者有權操作該網站的確認；敏感操作需有明確授權。

### Outputs

- 完成的網頁操作與使用者要求的檔案（截圖、HTML、JSON 等）。
- 實際使用的 profile／腳本／輸出路徑。
- `cloakbrowser info`、頁面標題或其他足以證明成功的驗證結果。

## Local Installation

目前本機基準狀態（每次仍應以實際檢查為準）：

- 原始碼：`D:\Github\cloakbrowser`
- npm CLI：`cloakbrowser`
- npm wrapper：`0.5.10`
- 免費 Chromium：`%USERPROFILE%\.cloakbrowser\chromium-146.0.7680.177.5\chrome.exe`
- skill runner：`scripts/cloak-runner.mjs`

檢查：

```powershell
cloakbrowser info
npm list -g --depth=0 cloakbrowser playwright-core
```

安裝或更新：

```powershell
npm install -g cloakbrowser@latest playwright-core@latest
cloakbrowser install
```

最新免費核心需要使用者自行完成 GitHub 登入；不要代替使用者處理憑證：

```powershell
cloakbrowser login
cloakbrowser update
```

## Procedure

1. 確認需求屬於 CloakBrowser，而非普通 Playwright 或接管現有 Chrome。
2. 執行 `cloakbrowser info --quick`；若套件或 binary 缺失才安裝。
3. 問清楚 headed/headless、是否保留 profile、輸出位置與代理需求。沒有指定時，一次性讀取用 headless；需要人工登入或觀察時用 headed。
4. 一般網站可複製 `assets/task-template.mjs` 到任務工作目錄並只修改副本；Shopee 不得直接使用該通用範本，因為它會在 `finally` 無條件關閉 browser。Shopee 任務必須採可在 challenge 狀態暫停、等待使用者明確確認後才續跑的兩階段生命週期；不要把一次性網站邏輯寫回 skill。
5. 從 skill 目錄執行 runner：

   ```powershell
   node scripts/cloak-runner.mjs C:\path\to\task.mjs
   ```

6. 對需要持久登入的任務，用 `launchPersistentContext` 並為每個用途使用獨立絕對路徑，例如 `D:\BrowserProfiles\cloakbrowser\<site-name>`。
7. 驗證 URL、標題、目標元素或輸出檔案；失敗時保存 screenshot 並回報實際錯誤。
8. 正常完成或確定停止後關閉 context/browser，不要留下無主 Chromium process；但 Shopee 若處於後述 `WAITING_FOR_USER` 狀態，必須依其專用生命週期保留同一 process/page/context，不能套用此通用關閉規則。

## Task Module Contract

`cloak-runner.mjs` 會載入全域安裝的 CloakBrowser，並執行任務模組的 default export：

```javascript
export default async function ({ cloak, args, skillDir }) {
  const browser = await cloak.launch({ headless: true });
  try {
    const page = await browser.newPage();
    await page.goto("https://example.com", { waitUntil: "domcontentloaded" });
    console.log(await page.title());
  } finally {
    await browser.close();
  }
}
```

Runner 的優點是不用在每個專案重複 `npm install`，也不依賴 ESM 不會讀取的 `NODE_PATH`。完整範本見 `assets/task-template.mjs`。

執行時可傳額外參數：

```powershell
node scripts/cloak-runner.mjs C:\work\task.mjs https://example.com C:\work\page.png
```

任務中可從 `args` 取得這些值。

## Recommended Patterns

### 一次性工作

用 `cloak.launch()`，並在 `finally` 中 `browser.close()`。

### 非無痕／保存登入狀態

```javascript
const context = await cloak.launchPersistentContext({
  userDataDir: "D:\\BrowserProfiles\\cloakbrowser\\example",
  headless: false,
  args: ["--fingerprint=42069"],
  humanize: true,
});
```

固定 seed 與固定 profile 應成對使用，避免同一網站每次呈現不同裝置身分。不要把 cookie、license key 或 proxy 密碼輸出到 log 或提交進 Git。

### 代理與地理設定

只有使用者提供或授權的代理才可使用：

```javascript
const browser = await cloak.launch({
  proxy: process.env.CLOAK_PROXY,
  geoip: true,
  headless: false,
  humanize: true,
});
```

`geoip: true` 需要 optional dependency `mmdb-lib`。缺少時再安裝：

```powershell
npm install -g mmdb-lib
```

### 截圖與證據

優先把輸出寫到任務工作目錄的絕對路徑：

```javascript
await page.screenshot({ path: outputPath, fullPage: true });
```

如果畫面或 selector 不符合預期，同時輸出目前 URL、title 與 screenshot，再判斷下一步，不要盲目重試或繞過 challenge。

## Rules and Limitations

- 僅操作使用者擁有、管理或明確獲准自動化的系統。
- CloakBrowser 的隱身、指紋或 humanize 功能不是 CAPTCHA／人機驗證的繞過工具，也不得用來規避 Shopee 或其他網站的偵測、封鎖、速率限制或安全機制。
- 不協助繞過 CAPTCHA、付費牆、登入／權限控制、封鎖、速率限制或網站安全機制；遇到滑塊、拼圖、驗證碼或其他 challenge 時，立即停止自動化，保留目前頁面並請使用者在瀏覽器中手動完成。只有使用者完成驗證後，才可繼續正常、低頻率且符合網站條款的操作。
- 不提供或執行代理輪換、指紋輪換、第三方 CAPTCHA 代解、破解驗證流程或其他以規避網站安全機制為目的的技術。
- 不建立批量假帳號、不進行 credential stuffing、不隱匿惡意活動。
- 不在 task、skill、shell history 或回覆中硬編碼／曝光 license key、cookie、帳密或代理密碼；使用環境變數或使用者自行登入。
- 任何發文、送出表單、付款、刪除、帳號變更等外部副作用，執行前都要確認目標與內容。
- CloakBrowser 是新的獨立 Chromium，不會自動繼承目前 Chrome 的登入狀態。
- 相對路徑以本 skill 目錄為基準；網站任務腳本與產出放在任務工作目錄，不污染正式 skill。
- 官方 binary 授權與 wrapper MIT 授權不同；遵守 `D:\Github\cloakbrowser\BINARY-LICENSE.md`。

## Shopee／蝦皮穩定模式

目標是降低因不一致工作階段、過度請求或錯誤重試造成的誤判，不保證不觸發網站風控，也不得將本流程用於繞過限制。

1. 只要目標網站是 `shopee.tw` 或使用者明確說「蝦皮／Shopee」，固定使用 CloakBrowser；不要呼叫一般 Chrome、Playwright MCP，也不要接管現有 Chrome。
2. 預設使用 **headed persistent context**，並長期重用同一個 Shopee 專用 profile，例如 `D:\\BrowserProfiles\\cloakbrowser\\shopee-tw`。同一 profile 固定搭配同一 fingerprint seed、語系、時區與正常網路出口；不得同時由兩個 browser/context 開啟。
3. 首次建立 profile、登入失效或網站要求驗證時，開啟可見視窗讓使用者自行登入／驗證。不得讀取、匯出或記錄使用者的 cookie、token、帳密。
4. 每次任務只啟動一個 context，原則上只使用一個 page，循序處理。直接完成使用者指定的最少查詢；不要平行大量開頁、快速翻頁、反覆 reload、輪換 IP／profile／fingerprint，或為同一失敗動作建立重試風暴。未另行約定時，單次任務預算為 1 個搜尋結果頁與最多 5 個商品詳情頁，相鄰頂層導航至少間隔 3 秒，且自動重試次數為 0；超過預算先停止並向使用者回報目前結果。
5. 導航與讀取以頁面狀態為準：使用 `domcontentloaded`、具體可見元素及合理 timeout；固定間隔只是節流下限，不可加入意圖偽裝成人類的隨機行為。商品清單只讀完成任務所需的最少欄位，已取得的結果保存在任務輸出中，避免重複請求。
6. 實作一個中央 `assertShopeeSafeState()` guard，至少檢查實際 URL、title、可見頁面文字，以及導航／主文件回應的 403、429。每次導航後、每次點擊或輸入前、每次資料擷取前都呼叫；不得把 challenge 錯誤交給一般 retry handler。
7. 若 URL 含 `/verify/`、`traffic/error`，或出現 CAPTCHA、滑塊、拼圖、驗證碼、安全性驗證等 challenge，guard 立即進入專用 `WAITING_FOR_USER` 狀態並停止所有自動操作。保持 headed 視窗、同一 page/context 與 task process 存活，通知使用者手動完成；不得點擊、求解、輪詢 challenge 元件或修改其請求。challenge 分支不得正常 return、throw 到 runner 頂層，或進入會關閉 context 的通用 `finally`。
8. 只有使用者明確表示已完成驗證後，才解除 `WAITING_FOR_USER`；先在同一 page/context 執行 guard 並重新檢查目標元素，只續跑原任務一次。若仍被攔截，不自動重試，改請使用者提供商品連結／搜尋結果，或改查官方及其他合規來源。
9. 對 HTTP 429、403、網路錯誤或頁面結構異常採 fail-closed：保存 URL、title、錯誤與必要截圖後停止。不要以換身分、換代理、清 cookie 或連續重試處理。429／403 不進入人工驗證等待流程，除非頁面本身明確呈現可由使用者完成的 challenge。
10. 正常結束時關閉 context，讓 persistent profile 正確落盤。等待人工驗證時，不得關閉、detach 或讓 runner 結束；執行環境若無法跨使用者回合保留 task process，就不要開始需要人工驗證後續的自動化，改為只開啟 headed 視窗並由使用者完成該階段。
11. 回報時說明使用 CloakBrowser、查詢時間、資料完整度、實際消耗的導航預算、profile 是否成功沿用，以及是否因人機驗證而有遺漏；不要宣稱「完全無法偵測」、「已繞過驗證」或保證下次不會被擋。

### Shopee 任務預設值

- `headless: false`
- `humanize: true`（僅用於正常輸入互動，不用於處理 challenge）
- 固定 Shopee 專用 `userDataDir`、fingerprint seed、`locale: "zh-TW"` 與 `timezone: "Asia/Taipei"`
- 啟動前確認 profile 未被其他 process 使用；沿用正常且一致的網路出口，不主動探測或記錄 IP
- 單一 context、單一 page、循序操作、最少必要導航
- 預設最多 1 個搜尋頁、5 個商品頁；頂層導航間隔至少 3 秒；零自動重試
- 不使用代理；只有使用者明確提供且有權使用時才設定，並避免在同一 profile 中任意改變地區
- 每項動作／擷取前執行中央 guard；驗證頁、403、429 一律 fail-closed

建議啟動骨架：

```javascript
const context = await cloak.launchPersistentContext({
  userDataDir: "D:\\BrowserProfiles\\cloakbrowser\\shopee-tw",
  headless: false,
  humanize: true,
  locale: "zh-TW",
  timezone: "Asia/Taipei",
  args: ["--fingerprint=42069"],
});

const pages = context.pages();
const page = pages[0] ?? await context.newPage();
page.setDefaultTimeout(30_000);
page.setDefaultNavigationTimeout(45_000);
```

固定 seed 只是維持同一個合法工作階段的環境一致性，不得輪換 seed 來規避封鎖。Shopee task 必須把 `RUNNING`、`WAITING_FOR_USER`、`STOPPED`、`COMPLETED` 當成不同狀態；只有 `STOPPED`／`COMPLETED` 可以進入關閉 context 的清理分支。`WAITING_FOR_USER` 必須以執行環境可跨回合維持的明確暫停／恢復機制保存同一 process/page/context，不能依賴通用範本的 `finally`。

## Pitfalls

- 全域 npm 套件是 ESM；直接從任意目錄 `import "cloakbrowser"` 常因模組解析而失敗。使用 bundled runner，或在專案本地安裝依賴。
- 不需要執行 `playwright install chromium`；CloakBrowser 下載自己的 Chromium binary。
- `launch()` 預設是一個一次性／類無痕 context；網站若需保存登入，改用 persistent context。
- Windows 不支援 README 所述的 Linux Widevine hint-file 流程。
- 不要假設 README 記載版本仍是最新；先執行 `cloakbrowser info` 與 `npm view cloakbrowser version`。
- `humanize` 只是輸入行為模擬，不代表可以無視網站授權、條款或 challenge。
- 不要聲稱「完全無法偵測」；實際結果受版本、IP、網站規則與配置影響。

## Verification

1. `cloakbrowser info --quick` 顯示 `Installed: true`。
2. 執行安全 smoke test：

   ```powershell
   node scripts/cloak-runner.mjs scripts/smoke-test.mjs
   ```

3. 輸出應包含 `title=Example Domain`、`url=https://example.com/` 與 `ok=true`。
4. 實際任務另驗證目標元素／資料或輸出檔案存在。
5. 回報 wrapper 版本、binary tier/version、任務結果與任何限制。
