---
name: open-design-install
description: 當使用者要求在 Windows 安裝、更新、啟動或驗證 nexu-io/open-design 的 source 版 OpenDesign 時使用；涵蓋官方 repository、Corepack/pnpm、build-script 核准、tools-dev、daemon/web 服務與安裝後驗證。
---

# OpenDesign source 安裝與執行

## When to Use

當使用者要：

- 安裝或更新官方 OpenDesign source repository。
- 依官方流程執行 `corepack enable`、`pnpm install`、`pnpm tools-dev run web`。
- 處理 Windows 上 `corepack enable` 無法寫入 `C:\Program Files\nodejs` 的 `EPERM`。
- 核准 pnpm 被忽略的 install/build scripts，例如 `@google/genai` 或 `node-pty`。
- 啟動、停止、檢查 OpenDesign source 的 web 與 daemon。

這個 Skill 管理的是 source checkout；已安裝的桌面版 OpenDesign 是另一個產品安裝，不要因為 source 安裝而覆蓋或移除它。

## Inputs and Outputs

### Inputs

- 官方來源：`https://github.com/nexu-io/open-design`
- 預設 source 路徑：`D:\Github\open-design`
- Windows、Node.js 約 24、pnpm 10.33.x。
- 可選的 daemon port；若未指定，`tools-dev` 可能動態分配 port。

### Outputs

- 完整的 OpenDesign source checkout 與 `node_modules`。
- 成功的 workspace postinstall/build 結果。
- 可核對的 web URL、daemon URL、版本與程序狀態。
- 安裝 log 一律寫到系統暫存目錄，不寫入 repository。

## Procedure

### Official source procedure

1. 先讀取並核對 repository 目前狀態，不要猜測路徑或覆蓋既有 checkout：

   ```powershell
   Test-Path D:\Github\open-design\package.json
   git -C D:\Github\open-design status --short --branch
   ```

2. 若目標不存在，執行官方 clone；若已存在，先確認 remote 是官方來源，通常使用 `git pull --ff-only` 更新。目標已存在且不是預期 repository 時停止。

3. 在 repository 根目錄執行官方安裝流程：

   ```powershell
   corepack enable
   pnpm install
   ```

4. Windows 若 `corepack enable` 因權限出現：

   ```text
   EPERM: operation not permitted, open 'C:\Program Files\nodejs\pnpx'
   ```

   不要為了繞過問題修改 Program Files 權限；改用：

   ```powershell
   corepack pnpm --version
   corepack pnpm install
   ```

   這是在同一個 Corepack 管理的 pnpm 上執行，避免依賴系統 PATH 裡不存在的 `pnpm` shim。

5. 官方 source quick start 是：

   ```powershell
   corepack pnpm tools-dev run web
   ```

   `run web` 是前景程序，會保留命令直到中斷。若需要由 tools-dev 管理背景服務，先查看當前狀態，再使用 repository 內已安裝的 binary：

   ```powershell
   corepack pnpm exec tools-dev status daemon
   corepack pnpm exec tools-dev start web --daemon-port 7456
   corepack pnpm exec tools-dev status daemon
   ```

   `--daemon-port 7456` 只是本機整合時的固定選項；web port 可由 tools-dev 動態分配，必須以實際輸出為準。

## pnpm build scripts

`pnpm install` 若顯示 `Ignored build scripts`，先向使用者說明套件與風險，再取得明確核准。核准後於 repository 根目錄執行：

```powershell
corepack pnpm approve-builds --all
corepack pnpm install
```

`approve-builds --all` 是本 repository 安裝依賴的授權，不代表可以在其他專案盲目核准所有套件。重新安裝後要確認原生模組的產物存在，例如 Windows `node-pty` 的 `conpty.dll` 與 `OpenConsole.exe`。

本機一次安裝時，核准動作將 `@google/genai` 與 `node-pty` 加入 root `package.json` 的 `pnpm.onlyBuiltDependencies`。這是 pnpm 的持久化設定變更，應在回報中明確列出，不要把它描述成 OpenDesign 原始碼功能修改。

## Windows command and PATH rules

- 優先使用 `corepack pnpm ...`，不要假設裸 `pnpm` 已經在背景 PowerShell 的 PATH 中。
- 優先使用 `corepack pnpm exec tools-dev ...`，避免 root script 內部再次解析不到 `pnpm`。
- 不要把 npm cache、pnpm store、`node_modules` 或 build output 複製到 `D:\OB\skills`。
- `corepack enable` 若需要系統管理員權限，不要自行提權或修改 ACL；改用 Corepack 直接執行，除非使用者另行授權系統層變更。

## Service lifecycle

### 啟動

官方開發流程：

```powershell
Set-Location D:\Github\open-design
corepack pnpm tools-dev run web
```

需要固定 daemon port、由 tools-dev 背景管理時：

```powershell
corepack pnpm exec tools-dev start web --daemon-port 7456
```

### 檢查

```powershell
corepack pnpm exec tools-dev status daemon
corepack pnpm exec tools-dev status web
Get-NetTCPConnection -State Listen -LocalPort 7456
```

HTTP 根路徑回傳 `404` 不代表 daemon 壞掉；應以 TCP listen、tools-dev status 與 OpenDesign API（例如 `/api/projects`）一起判斷。

### 停止

停止前要確認確實是本次 OpenDesign source 啟動的 namespace；不要用廣泛的 `taskkill` 殺掉不明 Node 程序：

```powershell
corepack pnpm exec tools-dev stop web
corepack pnpm exec tools-dev stop daemon
```

## Rules and Limitations

- 官方 source 流程以 repository README 為準：`corepack enable && pnpm install`，再 `pnpm tools-dev run web`。
- 本機目前的 source checkout 是 `D:\Github\open-design`，但 Skill 不得假設所有機器都使用這個路徑；腳本提供參數覆寫。
- 不覆蓋非空目標、不刪除既有桌面版、不改動無關 MCP server。
- daemon/web 啟動是常駐程序；執行前要清楚告知 port 與生命周期。
- 安裝腳本、測試 log 與暫存輸出放系統 `%TEMP%\pi-work\...`；只有正式 source、lockfile、`package.json` 的 pnpm 核准設定可寫入 repository。
- 不在 Skill 中保存 API key、OAuth token、cookie 或其他憑證。

## Pitfalls

- 在任意目錄執行 `pnpm tools-dev run web` 會得到錯誤或啟動錯誤專案；一定要先 `Set-Location` 到 source root。
- `corepack enable` 成功不代表目前背景程序一定能找到裸 `pnpm`；背景啟動仍使用 `corepack pnpm`。
- `pnpm approve-builds` 是互動命令；自動化時使用 `--all` 前必須已有使用者核准。
- `tools-dev run web` 顯示的 web port 與 daemon port 可能不同；不要把 web port 猜成 7456。
- 安裝完成但 `@google/genai`、`node-pty` 的 script 被忽略時，不可宣稱原生功能已完整驗證。
- source daemon 與桌面版 daemon 不要同時佔用相同 port；先查 listen socket 與 tools-dev status。

## Verification

1. 版本與 source：

   ```powershell
   node --version
   corepack pnpm --version
   git -C D:\Github\open-design remote get-url origin
   git -C D:\Github\open-design status --short --branch
   ```

2. 安裝產物：

   ```powershell
   Test-Path D:\Github\open-design\node_modules
   Test-Path D:\Github\open-design\tools\dev\dist\index.mjs
   ```

3. pnpm 核准設定：確認 `package.json` 的 `pnpm.onlyBuiltDependencies` 是否包含本次實際核准項目。

4. 服務：

   ```powershell
   corepack pnpm exec tools-dev status daemon
   Invoke-WebRequest http://127.0.0.1:7456/api/projects -UseBasicParsing
   ```

5. 任何測試 log 或中間檔案必須留在 `%TEMP%\pi-work\<task-id>\`，任務結束前確認沒有背景程序仍依賴它；不需要交付時清理。

## Bundled script

- `scripts/install-open-design.ps1`：安全地 clone／確認官方 remote、使用 Corepack 安裝依賴、可選核准 build scripts；預設不啟動常駐服務，也不覆蓋非空目標。

## Conformance Addendum

## Inputs and Outputs
- **Input:** 官方 OpenDesign repository、Windows source path、Node/Corepack/pnpm 狀態與目前 daemon 狀態。
- **Output:** 可重複的 source 安裝／驗證結果與實際 URL、port、版本。

## Rules and Limitations
- 相對路徑以本 Skill 資料夾為基準。
- 安裝前確認目標與 remote；安裝後列出 pnpm build-script 核准的持久化變更。
- 不執行未經使用者核准的第三方 build script；不暴露憑證。

## Pitfalls
- 不要把 `corepack enable` 的系統 shim 權限錯誤誤判成 pnpm install 失敗。
- 不要把 HTTP 根路徑 404 誤判成 daemon 未啟動。

## Verification
- YAML frontmatter 可解析，腳本通過 PowerShell parser 檢查。
- source `package.json`、`node_modules`、tools-dev build output 存在。
- `tools-dev status` 與 OpenDesign API 回應可核對。
