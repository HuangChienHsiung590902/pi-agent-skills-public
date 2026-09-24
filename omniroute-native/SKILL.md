---
name: omniroute-native
description: >-
  Manage the native Windows OmniRoute AI gateway at http://localhost:20128/v1
  and its dashboard. Use this skill whenever the user mentions OmniRoute,
  localhost:20128, ACP agents, concurrent LLMs, parallel requests, or
  `chat_admission_busy`/503 errors. Covers native installation and migration,
  safe start/stop/restart, npm native-module pitfalls, provider diagnostics,
  and configuring the global heavyweight-request concurrency limit.
---

# OmniRoute — Native (non-Docker) Install

## 為什麼要從 podman 容器搬到原生安裝

OmniRoute 的 `/dashboard/acp-agents`（ACP 代理）頁面靠「在 PATH 裡執行 `<cmd> --version`」偵測本機裝了哪些 CLI 代理（`claude`、`codex`、`cursor`…）。這個偵測是在 **OmniRoute 自己的程序裡**跑的——當年用 podman 容器跑（`docker.io/diegosouzapw/omniroute:latest`）時，容器內 PATH 是 Linux 環境，永遠看不到 Windows 主機上的 `claude.exe`（`C:\Users\HCH\.local\bin\claude.exe`），就算裝了也沒用（Windows exe 在 Linux 容器裡跑不起來）。ACP 是反向流架構（`Omni → spawn CLI → resp`），CLI 必須是 OmniRoute 程序能直接 spawn 的子行程，所以**必須跑在原生 Windows**，才可能偵測到 Windows-only 的 CLI（`cursor`、`warp` 等）。

npm 有發行原生版：套件名 `omniroute`（`bin/omniroute.mjs`），engines 需要 `node >=22.22.2 <23 || >=24.0.0 <27`，本機 node v24.18.0 符合。

## 資料遷移（容器 → 原生，一次性，已完成）

1. 乾淨停機容器讓 SQLite WAL 完整 checkpoint：`podman stop omniroute`（**不要 rm**，資料卷 `omniroute-data` 留著當回退點）
2. 複製資料到原生預設資料目錄：`podman cp omniroute:/app/data/. "$env:USERPROFILE\.omniroute"`
   - 內容：`storage.sqlite`（主資料庫）、`server.env`（JWT_SECRET/STORAGE_ENCRYPTION_KEY/API_KEY_SECRET，**這把加密金鑰不對就解不開既有 provider 憑證**）、`call_logs/`、`db_backups/`、`logs/`、`cloudflared/`
3. 原生 CLI 預設資料目錄就是 `~/.omniroute`（可用 `DATA_DIR` 環境變數覆蓋），啟動時偵測到 `server.env` 存在但 `.env` 不存在會自動搬移（`.env` 是實際讀取的檔名，`server.env` 是舊 Electron 格式的相容命名）——這是**預期行為**，log 會印 `♻ Migrated Electron secrets from ... to ...`，不是錯誤。

若需要回退：容器映像與 volume 都還在，`podman start omniroute` 即可復活舊容器（先確認原生服務已 `omniroute stop`，避免兩邊搶 port 20128）。

## npm 全域安裝的坑：`allow-scripts` 不支援 global，`npm link` 繞過

**現象**：`npm install -g omniroute` 會成功裝完，但終端機印出一堆
```
npm warn allow-scripts 11 packages have install scripts not yet covered by allowScripts:
npm warn allow-scripts   omniroute@3.8.49 (postinstall: node scripts/build/postinstall.mjs)
npm warn allow-scripts   keytar@7.9.0 (install: ...)
npm warn allow-scripts   sharp@0.34.5 / esbuild / better-sqlite3(經由 omniroute postinstall) / onnxruntime-node / koffi ...
```
這是新版 npm（本機 11.16.0）的供應鏈安全機制，**預設擋掉所有原生模組（native addon）的 build/install 腳本**，要求逐一批准。`omniroute` 自己的 postinstall（`scripts/build/postinstall.mjs`）**不是可有可無**——npm 發行包裡 standalone build 內建的 `better-sqlite3`/`wreq-js`/`tls-client-node` 原生綁定是照 **Linux x64**（打包機平台）編譯的，這支腳本負責把 npm 已經正確編譯好的 Windows 版原生綁定複製進去覆蓋掉；不跑的話 SQLite 在 Windows 上會直接壞掉。

**死路**：`npm approve-scripts --allow-scripts-pending -g` / `npm approve-scripts <pkg> -g` 一律回傳
```
npm error code EGLOBAL
npm error `npm approve-scripts` does not work for global installs
```
這個 gating 機制**完全不支援 `-g`**，不是設定問題，沒有旗標能繞。

**解法**：改在一個有 `package.json` 的本機資料夾裡裝成 local dependency（`npm approve-scripts` 需要專案上下文才能運作），批准腳本，再用 `npm link` 把它接回全域 PATH：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.omniroute-install" | Out-Null
Set-Location "$env:USERPROFILE\.omniroute-install"
npm install omniroute          # npm 會自動生成最小 package.json，npm init 失敗也沒關係
npm approve-scripts --all      # 批准並實際執行全部待批准的原生模組腳本（含 omniroute 自己的 postinstall）

Set-Location "$env:USERPROFILE\.omniroute-install\node_modules\omniroute"
npm link --ignore-scripts      # --ignore-scripts 必加：這裡直接 link 套件目錄本身，
                                # npm 會誤觸發它的 "prepare" 腳本（跑 husky，
                                # 那是原始 repo 給貢獻者裝 git hook 用的開發者工具，
                                # 對「被裝成相依套件」的情境完全用不到、也裝不了）
```

完成後全域 `omniroute` 指令（`npm root -g` 底下的 `omniroute.cmd`/`.ps1` shim）會是個 junction，實際指向 `C:\Users\HCH\.omniroute-install\node_modules\omniroute`——這個資料夾底下的原生模組才是真的修好、可以在 Windows 上跑的版本。**不要事後刪除 `.omniroute-install` 資料夾**，全域指令靠它才能動。

驗證原生模組是否正確：`omniroute --version` 能印出版本號只代表 CLI 入口能跑，不代表 SQLite 綁定是對的；用 `omniroute status` 才會真正打開資料庫（`Database: Found (x MB)`）跟列出 CLI Tools 偵測結果，這才是完整驗證。


## 安裝 GitHub release 分支（未發布到 npm 的版本，例如 release/v3.8.51）

當使用者貼 GitHub 分支 URL（例如
`https://github.com/diegosouzapw/OmniRoute/tree/release/v3.8.51`）要求安裝，**不要直接用**
`npm install -g omniroute`：npm registry 可能只發布到較舊版本（實測 v3.8.51 當時 npm 只到
`3.8.49`）。正確做法是從 source build 出正式 tarball，再全域安裝。

本 skill 已提供可重用腳本：

```powershell
D:\OB\skills\omniroute-native\scripts\install-github-release.ps1 `
  -Branch release/v3.8.51 `
  -RepoDir D:\Github\OmniRoute `
  -StartDaemon
```

### 手動流程（腳本做的事）

1. 確認版本來源：
   ```bash
   git ls-remote --heads https://github.com/diegosouzapw/OmniRoute.git | grep release/v3.8.51
   npm view omniroute version
   npm view omniroute@3.8.51 version   # 若 404，代表必須 source build
   node -v                             # 需符合 package.json engines
   ```
2. Clone/update source：
   ```bash
   git clone --depth 1 --branch release/v3.8.51 https://github.com/diegosouzapw/OmniRoute.git D:/Github/OmniRoute
   cd D:/Github/OmniRoute
   ```
3. 安裝 dependencies，遇到 npm 11 `allow-scripts` warning 時在專案上下文批准並重建必要原生套件：
   ```bash
   npm install --no-audit --no-fund
   npm approve-scripts --all
   npm rebuild tls-client-node koffi libxmljs2 onnxruntime-node @playwright/browser-chromium opencode-ai
   ```
   `bun` rebuild 可能因 `@oven/bun-windows-x64` optional binary 缺失而失敗；對 OmniRoute server 核心功能不是必要，可先跳過。
4. 套用 v3.8.51 Windows/source-pack 兩個本地修補：
   - `package.json` 的 `files[]` 必須移除 `"!**/node_modules/**"`。否則 `npm pack` 會把
     `dist/node_modules` 整個排除，安裝後 standalone server 會缺
     `better-sqlite3`、`tls-client-node`、`onnxruntime-node`、`wreq-js` 等 native runtime。
     對比：官方 `3.8.49` tarball 有 `dist/node_modules/`，而未修補的 v3.8.51 pack 會是 0 個。
   - `scripts/build/prepublish.ts` 中 ChatGPT Web Codex MCP bridge 若仍是：
     `execFileSync(NPX_BIN, ["esbuild", ...])`，改用同檔已存在的
     `runBuildTool("esbuild", "esbuild", [...])`。Windows + Node 24 直接 spawn `npx.cmd`
     會因 Node 的 `.cmd` hardening 出現 `spawnSync npx.cmd EINVAL`。
5. Build/publish artifact：
   ```bash
   cd D:/Github/OmniRoute
   rm -rf .build dist
   export OMNIROUTE_BUILD_SHA=$(git rev-parse --short HEAD)
   npm run build
   npm run build:cli
   node scripts/build/write-build-sha.mjs
   npm pack
   ```
   Windows 上 repo 原本的 `npm run build:release` 內含 bash 語法
   `OMNIROUTE_BUILD_SHA=$(...) npm run build`，npm 會交給 `cmd.exe` 執行而失敗；要像上面分段跑。
6. 驗證 tarball **必須**包含 native runtime：
   ```bash
   tar -tzf omniroute-3.8.51.tgz | grep -c "dist/node_modules/"     # 應遠大於 0
   tar -tzf omniroute-3.8.51.tgz | grep "dist/node_modules/better-sqlite3/prebuilds/win32-x64.node"
   ```
7. 全域安裝並驗證：
   ```bash
   npm install -g ./omniroute-3.8.51.tgz --no-audit --no-fund
   omniroute --version        # 應印 3.8.51
   omniroute serve --daemon --no-open
   omniroute health --timeout 15000
   omniroute models --timeout 15000
   ```

### Windows build/pack 踩坑

- Everything、Windows Search、Defender 可能在 `dist/` 大量檔案剛產生時鎖檔，導致
  `EPERM`、`Device or resource busy`、或 build 後續工具莫名失敗。先把 `dist` rename 成
  `dist_old_<timestamp>`，再用 `cmd /c rd /s /q dist_old_<timestamp>` 清掉，比 git-bash
  `rm -rf dist` 穩。
- `omniroute --version` 只驗證 CLI entry，不代表 standalone native runtime 完整。必須再檢查：
  ```bash
  ROOT="$(npm root -g)/omniroute"
  test -e "$ROOT/dist/node_modules/better-sqlite3/prebuilds/win32-x64.node"
  test -e "$ROOT/dist/node_modules/tls-client-node/bin/tls-client-windows-64-1.15.1.dll"
  test -e "$ROOT/dist/node_modules/onnxruntime-node/bin/napi-v6/win32/x64/onnxruntime_binding.node"
  test -e "$ROOT/dist/node_modules/@wreq-js/binding-win32-x64-msvc/wreq-js.win32-x64-msvc.node"
  ```
- `omniroute serve --daemon --no-open` 可能需要 1–2 分鐘才完成所有背景 sync；若 `/api/health`
  暫時沒回，先看 `~/.omniroute/logs/application/app.log`，再用 `omniroute health` 判斷。
- `/v1/models` 回 `Authentication required` 不是安裝壞掉，代表目前 server 要求有效的
  gateway API key；用 `omniroute models` 只能作為 CLI catalog 輔助檢查，不能取代
  Claude Code 所需的 authenticated `GET /v1/models` 測試。
- Claude Code 的 gateway model discovery 需要同時具備
  `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1`、Anthropic gateway root
  `ANTHROPIC_BASE_URL=http://localhost:20128`（不要加 `/v1`）、與 OmniRoute
  接受的 `ANTHROPIC_AUTH_TOKEN`。要讓非 Claude 模型以 `claude/<provider>/<model>` alias
  被 discovery 看見，另外確認 `EXPOSE_CC_DISCOVERY_ALIASES=1` 或等效 feature flag。
  只看到 discovery flag 或只看到 `/api/health` 正常，都不能證明 catalog discovery 已經成功。
- `/v1/models` 回 401 時，先區分 `Authentication required`（沒有有效 Bearer token）與
  `Invalid API key`（token 不再是 active gateway key）；不要先修改模型 ID 或宣稱 Claude
  Code 不支援 OmniRoute。
- v3.8.51 的 MITM utilities typecheck 可能出現 `TS6059` / `TS2307`，目前 `prepublish.ts` 標示為
  non-fatal warning；核心 dashboard、OpenAI-compatible `/v1`、providers/models 不依賴這一步。

## 啟動 / 停止 / 重啟

服務指令是 `omniroute serve`（`serve` 是預設動作，直接打 `omniroute` 也會啟動）。背景啟動（PowerShell 的 `Start-Process -FilePath 'omniroute.ps1'` 會噴 `%1 不是有效的 Win32 應用程式`，`.ps1`/`.cmd` shim 不能直接當 exe 跑，要包一層 `cmd.exe /c`）：

```powershell
Start-Process -FilePath 'cmd.exe' -ArgumentList '/c', "$(npm root -g)\..\omniroute.cmd serve > $env:TEMP\omniroute-serve.log 2>&1" -WorkingDirectory "$env:USERPROFILE" -WindowStyle Hidden
```

**停止：一定要用 `omniroute stop`，不要 `taskkill` port 20128 上的 PID。** 服務內建一個監控/supervisor 會自動重啟被殺掉的 worker 子行程（`server-ws.mjs`）——實測 `taskkill /F /PID <port 20128 的 PID>` 之後，幾秒內 port 20128 又被一個**新 PID** 佔用，換了進程但服務沒真的停。正確做法：

```powershell
$root = npm root -g
& "$root\..\omniroute.cmd" stop      # 印出 "Stopping server (PID xxxx)... Server stopped." 才是真的停了
```

重啟 = `stop` 之後再用上面的 `Start-Process` 背景啟動一次。**改密碼、改 `.env`、清資料庫任何內容之後都必須重啟才會生效**（reset-password 腳本本身會提示 `Restart OmniRoute for changes to take effect.`）。

## 多個 LLM / 多個 request 同時執行

### 症狀

OmniRoute 回傳：

```text
503 chat_admission_busy
Chat admission capacity is temporarily unavailable. Retry shortly.
```

但需求是同時執行多個模型或多個 agent。這通常不是 provider 不支援相同模型並行，而是 OmniRoute 3.8.49+ 的 heavyweight chat admission 在入口先限制了大型 request。

### 版本差異

- **3.8.48**：主要在 V8 heap 有壓力時才 shed 大型 body；正常 heap 狀態下不會以固定 heavy-request lease 擋住並發。
- **3.8.49 / 3.8.50**：新增 heavyweight admission；預設 `OMNIROUTE_CHAT_MAX_HEAVY_IN_FLIGHT=1`。多個 coding-agent request 會被判定為 heavyweight，第二個 request 等待後可能收到 `503 chat_admission_busy`。
- 目前已查證同一個模型、甚至同一個 Codex connection，仍可存在重疊成功請求；因此要先區分 admission gate、connection `max_concurrent` 與 provider 真實 rate limit。

### 建議設定

在使用者資料目錄的 `.env`（Windows 預設為 `$env:USERPROFILE\.omniroute\.env`；實際操作前先用 `omniroute status` 確認 Data Dir）加入或更新：

```env
# 所有模型合計的 heavyweight request 並發名額
OMNIROUTE_CHAT_MAX_HEAVY_IN_FLIGHT=8

# 健康 heap 快速路徑的額外並發名額
OMNIROUTE_CHAT_ADMISSION_HEALTHY_HEADROOM=8

# 名額滿時排隊等待時間，避免 agent 很快耗盡 retry 次數
OMNIROUTE_CHAT_ADMISSION_QUEUE_MS=30000
```

這代表所有模型共用最多 8 個 heavyweight request，不是每個模型各 8 個；可以混合執行不同模型，也可以同時執行同一模型的多個 request。超過 8 個時最多等待 30 秒，仍無容量才回 503。

### 套用流程（Windows native install）

1. **先告知並確認**：修改 `.env` 並重啟會中斷正在執行的 request；未經使用者確認，不要寫入或重啟。
2. 備份：
   ```powershell
   $p = "$env:USERPROFILE\\.omniroute\\.env"
   Copy-Item $p "$p.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
   ```
3. 用 `omniroute stop` 停止，不要用 `taskkill`：
   ```powershell
   omniroute stop
   ```
4. 透過 `cmd.exe /c` 背景啟動，避免 Windows shim 被 `Start-Process` 當成 executable：
   ```powershell
   Start-Process -FilePath 'cmd.exe' -ArgumentList '/c', "$(npm root -g)\\..\\omniroute.cmd serve > $env:TEMP\\omniroute-serve.log 2>&1" -WorkingDirectory "$env:USERPROFILE" -WindowStyle Hidden
   ```
5. 驗證：
   ```powershell
   omniroute status
   curl.exe http://127.0.0.1:20128/api/health
   Get-NetTCPConnection -State Listen -LocalPort 20128
   ```

### 排錯順序

1. 先查 `app.log` 是否有 `module=chat-admission`、`reason=queue_timeout`、`activeHeavy=1` 或 `chat_admission_busy`。
2. 查 `.env` 是否真的有設定，並確認重啟後的新 server process 已載入；模組在 process 啟動時讀取環境變數，熱改檔案不會生效。
3. 若仍有 503，確認是不是超過全域 8 個名額、`OMNIROUTE_CHAT_ADMISSION_MAX_QUEUED_BYTES` 是否太小，或 V8 heap／實體記憶體真的有壓力。
4. 再檢查 provider connection 的 `max_concurrent`、provider API concurrency 與 rate limit；不要把 `chat_admission_busy` 誤判成模型本身不支援並行。
5. 不要直接把名額無限放大；heavy request 會在解析、翻譯、壓縮和 dispatch 期間產生多份暫存資料，過高可能再次造成 OOM。

### 本機目前已知狀態

本機曾套用上述設定並驗證：

```text
OMNIROUTE_CHAT_MAX_HEAVY_IN_FLIGHT=8
OMNIROUTE_CHAT_ADMISSION_HEALTHY_HEADROOM=8
OMNIROUTE_CHAT_ADMISSION_QUEUE_MS=30000
```

當時新 server process 成功監聽 `20128`，`/api/health` 回傳 `{"status":"ok"}`。

2026-09-21 再查：套件 `.env` 裡這三個鍵仍是註解預設 `1`；**必須寫進** `C:\Users\Administrator\.omniroute\.env` 才會生效。當時使用者資料目錄的 `.env` 沒有這三鍵，等於仍是 1 路。已備份 `.env.bak-20260921083741`、追加三鍵、`omniroute stop` 後用 `cmd.exe /c omniroute.cmd serve --no-open` 重啟，health 為 ok。這是 OmniRoute 入口閘門，**與 llama.cpp `-np`／n_slots 無關**；遠端 30B 容器仍是 `n_slots=4`。

### 實測紀錄：Windows 原生安裝（Administrator，2026-09-18）

這次實際套用並確認的設定檔是：

```text
C:\Users\Administrator\.omniroute\.env
```

操作流程與結果：

1. 先將原檔備份為同目錄下的 `.env.bak-YYYYMMDD-HHMMSS`。
2. 若參數已存在，更新原值；若不存在，追加到檔案末尾，避免產生重複鍵。
3. 執行 `omniroute stop`，不要直接 `taskkill` port 20128 的 PID。
4. Windows 背景啟動不要直接把 `.cmd` shim 當成 executable；使用 `cmd.exe /c` 啟動：
   ```powershell
   $cmd = Join-Path $env:APPDATA 'npm\omniroute.cmd'
   $log = Join-Path $env:TEMP 'omniroute-serve.log'
   Start-Process -FilePath 'cmd.exe' `
     -ArgumentList '/c', "`"$cmd serve --no-open > `"$log`" 2>&1`"" `
     -WorkingDirectory $env:USERPROFILE -WindowStyle Hidden
   ```
5. 以 `http://127.0.0.1:20128/api/health` 驗證，實測回傳：
   ```json
   {"status":"ok"}
   ```

注意：啟動程序可能先成功監聽 port，但健康端點仍需等待背景初始化；不要只因第一次 health request timeout 就判定啟動失敗，應查看 `$env:TEMP\omniroute-serve.log` 並重試。

### Claude Code gateway discovery 設定與驗證

若要讓 Claude Code 讀取 OmniRoute 的所有目前可用模型，除了 Claude Code 的設定外，OmniRoute 使用者資料目錄的 `.env` 還要啟用非 Claude 模型 alias：

```env
EXPOSE_CC_DISCOVERY_ALIASES=1
```

設定檔位置是 `$env:USERPROFILE\.omniroute\.env`，不是 npm 套件目錄內的 `.env`。修改後必須完整重啟；熱改檔案不會更新已存在的 server process。

安全流程：

```powershell
$p = "$env:USERPROFILE\.omniroute\.env"
Copy-Item $p "$p.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
omniroute stop
# 以 cmd.exe /c 啟動 omniroute.cmd，避免直接執行 .cmd/.ps1 shim
Start-Process -FilePath 'cmd.exe' -ArgumentList '/c', "`"$env:APPDATA\npm\omniroute.cmd`" serve --no-open" -WorkingDirectory $env:USERPROFILE -WindowStyle Hidden
```

重啟後分開驗證：

```powershell
omniroute health
$models = Invoke-RestMethod `
  -Uri "http://127.0.0.1:20128/v1/models" `
  -Headers @{ Authorization = "Bearer $env:OMNIROUTE_API_KEY" }
$models.data.Count
@($models.data | Where-Object { $_.id -like "claude/*" }).Count
```

`health` 正常只代表服務存活，不代表 Claude Code 的 Bearer token 有效；`/v1/models` 回 HTTP 200 才能證明 discovery catalog 可讀取。不要在命令列、log 或回覆輸出完整 token。

### Dashboard 模型選擇器：固定只顯示已設定 provider

封鎖 no-auth provider 後，其他未設定 provider 的模型仍可能出現在 Dashboard 模型選擇器。若需求是只顯示有有效 provider connection 的模型，使用 `omniroute-noauth-providers` Skill 的 configured-only Dashboard chunk 補丁：

- 搜尋包含 `modelSelectShowConfiguredOnly` 的 `.build/next/static/chunks/*.js`，不要假設 chunk 檔名永久固定。
- 修改前先備份成 `.bak-configured-only-YYYYMMDDHHMMSS`。
- 將 configured-only 的初始狀態固定為 `true`，並將 checkbox 固定為 checked/disabled。
- 只修改前端顯示層，不刪除 provider connection、SQLite 模型資料或 `/v1/models` catalog。
- 從 Dashboard HTTP 重新取得該 chunk，確認實際回應包含補丁；再讓使用者 `Ctrl+F5`。

此類 static chunk 補丁通常不需要改資料庫，但若服務剛升級或重啟後仍送出舊 chunk，應先檢查安裝路徑、manifest、瀏覽器快取及目前 server process 使用的 build，再決定是否依安全流程重啟。OmniRoute npm/source 升級會覆蓋 chunk，升級後必須重新檢查。

## 重設管理密碼

套件另外發行一支獨立工具 `bin/reset-password.mjs`（全域 shim 是 `omniroute-reset-password`，但透過 `npm link` 只 link 了 `omniroute` 主指令，這支要直接用 `node` 呼叫）。它是互動式的（兩次輸入確認），非互動情境（腳本/被 agent 呼叫）改用 stdin pipe，**新密碼須 ≥ 8 字元**：

```powershell
$root = npm root -g
$pwd_ = "你的新密碼"
"$pwd_`n$pwd_`n" | node "$root\omniroute\bin\reset-password.mjs"
```

跑完務必 `omniroute stop` + 重新背景啟動（見上一節），否則舊 session 還在跑用的是舊密碼的驗證邏輯。

## Provider Topology（提供者拓撲）出現不會消失的紅色錯誤節點

**現象**：`/dashboard/providers`（拓撲圖）某個 provider 節點（例如 "Anthropic"）持續顯示紅色、「0 有效 · 1 錯誤」，但實際上 `omniroute providers list --json` 查出來所有真正在用的連線都是 `isActive: true` / `testStatus: "active"`，日誌裡真實的 API 呼叫（`/v1/messages` 之類）也全部 `status:"success"`。

**根因**：這張圖（`src/lib/monitoring/providerHealthMatrix.ts`）不是直接讀 provider connection 的健康狀態，而是**照 `call_logs` 資料表的 `provider` 欄位字串分組**、統計最近 range（預設 24h）內每個 provider/connection 的請求成敗，畫出獨立節點。如果曾經建過一個 provider 連線、後來又刪掉了，但它產生過的舊 `call_logs` 記錄還在資料庫裡、時間還落在 24h 視窗內，這些記錄的 `connection_id` 已經對不到任何現存連線 → 程式碼把它當成「synthetic / Unattributed traffic」帳號單獨列一個 provider 節點，只要那批舊記錄裡有任何一筆失敗（`status>=400` 或 `error_summary` 不是 null），這個幽靈節點就會被判定為 degraded/error，並持續顯示到那些記錄自然超過 24h 視窗為止。

**注意 provider 字串可能對不上**：診斷時發現真正在用的連線 `provider` 欄位是 `"claude"`（OAuth 帳號），但幽靈記錄的 `provider` 欄位是 `"anthropic"`（不同字串、不同分組）——不要看到「都是 Anthropic 相關」就假設是同一批資料,要直接查 DB 比對 `provider` + `connection_id` 兩個欄位。

**排查步驟**：
1. 用日誌反查是哪個 connection_id 在報錯：`Select-String`/`grep` app.log（`~/.omniroute/logs/application/app.log`）找 `CredentialHealth`/`error` 相關行，記下 connection_id
2. 確認這個 connection_id 是否還存在於現役連線：`omniroute providers list --json`（若查無此 ID，就是孤兒）
3. 直接查 DB 佐證（server 可以繼續開著，這步是唯讀）：
   ```bash
   node -e "
   const Database = require('better-sqlite3');
   const db = new Database('C:/Users/HCH/.omniroute/storage.sqlite', { readonly: true });
   console.log(db.prepare(\"SELECT provider, connection_id, COUNT(*) n, MAX(timestamp) last, SUM(CASE WHEN status>=400 OR status<200 OR error_summary IS NOT NULL THEN 1 ELSE 0 END) errors FROM call_logs WHERE timestamp >= datetime('now','-1 day') GROUP BY provider, connection_id ORDER BY last DESC\").all());
   "
   ```
   （用 `.omniroute-install\node_modules\omniroute` 底下裝好的 `better-sqlite3`，`cd` 到那個資料夾再跑，才能吃到正確編譯的原生模組）
4. 確認是孤兒資料後清除：**先 `omniroute stop`**（避免跟 server 自己的 sqlite 連線互搶寫入），用有寫入權限的連線（不加 `readonly`）執行：
   ```js
   db.prepare("DELETE FROM call_logs WHERE connection_id = ?").run('<孤兒 connection_id>');
   ```
   刪完重新 `serve` 啟動。

**另一個會誤觸發同一症狀的操作**：`omniroute providers test <id>` 這個 CLI 指令**只認得 API-key 型連線**，對 OAuth 型連線（Claude 訂閱帳號登入）跑會回 `FAIL ...: Connection ... is not an API-key provider.`——這個失敗訊息會被寫回該連線的 `testStatus`/`lastError` 欄位，讓拓撲圖瞬間變紅，但連線本身完全沒壞。系統背景有一個 `CredentialHealth` 排程（開機後 30s 延遲、之後每 300s 一輪，會用正確方式重新驗證所有連線），**5 分鐘內會自動蓋回正確狀態**，不用手動修——但診斷時不要手滑對 OAuth 連線跑 `providers test`，會製造出跟真正故障一模一樣的假象，白繞一圈。

## 關鍵路徑一覽

| 內容 | 路徑 |
|---|---|
| 資料目錄（DB、金鑰、log） | `$env:USERPROFILE\.omniroute`（目前實機為 `C:\Users\Administrator\.omniroute`） |
| 主資料庫 | `$env:USERPROFILE\.omniroute\storage.sqlite` |
| 應用程式日誌 | `$env:USERPROFILE\.omniroute\logs\application\app.log` |
| 實際安裝的套件（原生模組修好的那份） | 依 `npm root -g` 與目前安裝方式確認，不要硬編使用者名稱 |
| 全域指令 shim | `%APPDATA%\npm\omniroute.cmd` / `.ps1`（junction 指向上面那份） |
| Dashboard | http://localhost:20128 |
| OpenAI 相容端點 | http://localhost:20128/v1 |
| 舊 podman 容器（保留，未刪，回退用） | `podman` container name `omniroute`, image `docker.io/diegosouzapw/omniroute:latest`, volume `omniroute-data` |

## Qwythos vLLM provider 與 pi 快取同步注意

本機 OmniRoute 有接遠端 `10.145.119.19:8182` 的 vLLM
`qwythos-9b-v2-awq`。目前遠端 vLLM 應回：

```bash
curl http://10.145.119.19:8182/v1/models
# max_model_len: 98304
```

pi 透過 OmniRoute 使用這顆模型時，不只看 OmniRoute provider，也會用
`D:\.system\.pi\agent\models.json` 的模型 metadata。若 `/omni sync` 或
OmniRoute 模型同步把 metadata 蓋回去，記得確認 qwythos 兩個 id：

- `qwythos/qwythos-9b-v2-awq`
- `openai-compatible-chat-dfdac666-5d49-4585-a3d4-210e5f8613e0/qwythos-9b-v2-awq`

都要是：

```json
"contextWindow": 98304,
"maxTokens": 8192
```

`D:\.system\.pi\agent\settings.json` 也要有：

```json
"reserveTokens": 8192
```

否則 pi 可能仍送 `max_tokens: 16384`，導致 vLLM 報 context overflow
（典型：`input 81921 + output 16384 = 98305 > 98304`）。

---

## Conformance Addendum

## When to Use
Manage the native (non-Docker) OmniRoute install on this Windows machine — an AI gateway/router (290+ providers, OpenAI-compatible endpoint at http://localhost:20128/v1, dashboard at http://localhost:20128) migrated from a podman container to native `npm i -g omniroute` so its ACP agent detector (/dashboard/acp-agents) can see Windows-native CLI binaries like claude.exe on PATH. Covers the docker-to-native migration, starting/ stopping/restarting the server (plain taskkill does NOT work — must use `omniroute stop`), the npm allow-scripts global-install trap and its `npm link` workaround, resetting the admin password, Claude Code gateway discovery aliases, and diagnosing "phantom" red/error nodes on the Provider Topology (提供者拓撲) page caused by orphaned call_logs rows referencing deleted provider connections. Use when the user mentions OmniRoute, localhost:20128, ACP agents / ACP 代理 detection, ACP 代理未找到, ACP CLI 偵測, or Claude Code discovery aliases.

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
