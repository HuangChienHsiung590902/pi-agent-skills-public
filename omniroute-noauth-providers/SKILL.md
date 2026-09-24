---
name: omniroute-noauth-providers
description: >-
  封鎖、解除或隱藏 OmniRoute Dashboard「免鑑權提供者」（no-auth / NOAUTH_PROVIDERS）整區。
  這些卡片是內建 catalog，沒有 provider_connections 列，不能用 providers remove 刪掉；
  官方機制是 settings.blockedProviders。涵蓋 13 個 id 對照、loopback CLI token 呼叫
  PATCH /api/settings、以及可選的 dashboard 前端補丁（沒有可見卡片就不渲染整區，
  升級 OmniRoute 會被蓋掉）。當使用者說「刪掉免鑑權提供者」「藏掉免鑑權」
  「Disabled 區塊不要出現」「block no-auth / OpenCode Free / DuckDuckGo AI Chat」
  時使用。不要動已設定的 Codex OAuth、Qwen、xAI、DeepSeek 連線。
---

# OmniRoute 免鑑權提供者：封鎖與隱藏

## When to Use

在下列情況載入本 Skill：

- 使用者要刪掉、停用、封鎖或藏起 OmniRoute「免鑑權提供者」。
- Dashboard `/dashboard/providers` 出現 AI Horde、OpenCode Free、DuckDuckGo AI Chat、Felo 等無需憑證卡片。
- 封鎖後畫面仍留 Disabled 區塊，要求整區不要顯示。
- 升級 OmniRoute 後免鑑權區又出現，要重套補丁。

不要用本 Skill 處理：

- OmniRoute 安裝／啟停／密碼 → `omniroute-native`
- 用 pi 呼叫 `oc/`、`ddgw/` 等免費模型 → `pi-omni-model-test`
- 內嵌服務 Bifrost／Mux → `omniroute-embedded-services`

## Inputs and Outputs

**Input**

- 本機 OmniRoute（`http://127.0.0.1:20128`）
- 設定庫 `D:\.system\.omniroute\storage.sqlite`（`C:\Users\HCH\.omniroute` 是 junction）
- 使用者要的是「只封鎖路由」還是「連畫面整區都不出現」

**Output**

- `blockedProviders` 含全部 no-auth id（或已解除）
- `/v1/models` 沒有 `oc/`、`ddgw/`、`felo/` 等前綴
- 若有套 UI 補丁：Providers 頁不再畫「免鑑權提供者」標題／Disabled 卡片
- 既有付費／OAuth 連線不變

## Procedure

先讀 `references/noauth-provider-ids.md`。這些卡片寫死在 catalog，**不能從軟體刪除**。

### 1. 只封鎖（官方、可逆、升級仍在）

```powershell
python D:\OB\skills\omniroute-noauth-providers\scripts\block_noauth.py status
python D:\OB\skills\omniroute-noauth-providers\scripts\block_noauth.py block
```

腳本會：

1. 用本機 machine-id 算出 loopback CLI token（不印出）
2. 從 `omniroute providers available --json --category noauth` 取現況 id
3. `PATCH /api/settings` 把那些 id **併入**既有 `blockedProviders`（不覆蓋其他封鎖）
4. 不碰 `provider_connections`

解除：

```powershell
python D:\OB\skills\omniroute-noauth-providers\scripts\block_noauth.py unblock
```

只從 `blockedProviders` 拿掉 no-auth id，其他封鎖保留。

### 2. 連畫面整區都不顯示（可選、升級會掉）

官方設計（#5166 / #5183）封鎖後仍顯示 Disabled + Enable。沒有設定開關能藏整區。

```powershell
python D:\OB\skills\omniroute-noauth-providers\scripts\hide_noauth_ui.py apply
```

條件改成：只有還有**可見**免鑑權卡片時才渲染 `NoAuthProvidersSection`。

還原官方 Disabled 區塊：

```powershell
python D:\OB\skills\omniroute-noauth-providers\scripts\hide_noauth_ui.py restore
```

套用後請 Ctrl+F5 刷新 `http://127.0.0.1:20128/dashboard/providers`。

### 3. 模型選擇器只顯示已設定 provider 的模型

封鎖 no-auth provider 只會移除那些 provider；其他未設定的 provider catalog 仍可能很多。OmniRoute 的 `ModelSelectModal` 有 `Show configured only` 篩選，但預設值存在瀏覽器 `localStorage`，可能仍然是關閉狀態。

若使用者要求模型瀏覽器**固定只顯示已設定的 provider**，可在已安裝的 Dashboard chunk 套用顯示層補丁：

1. 找到包含 `modelSelectShowConfiguredOnly` 的 `.build/next/static/chunks/*.js`。
2. 修改前先以 `.bak-configured-only-YYYYMMDDHHMMSS` 備份。
3. 將 `useState(() => "true" === localStorage.getItem("modelSelectShowConfiguredOnly"))` 改為 `useState(() => true)`。
4. 將 checkbox 改成固定 checked 並 disabled，避免使用者關閉篩選。
5. 不修改 provider_connections、模型資料庫或 `/v1/models` 原始 catalog。
6. 由 Dashboard HTTP 實際重新取得同一 chunk，確認回應已包含補丁。

目前本機 OmniRoute 3.8.50 的實際安裝位置是：

```text
C:\Users\Administrator\AppData\Roaming\npm\node_modules\omniroute\dist\.build\next\static\chunks\33_1-slzsge6t.js
```

備份與安裝路徑可能隨版本改變；不要硬編 chunk 檔名。套用後重新整理 Dashboard（`Ctrl+F5`）。此補丁只影響 Dashboard 模型選擇器的顯示，不會刪除可由完整 model ID 直接呼叫的 catalog 項目；升級 OmniRoute 後可能被覆蓋，必須重新檢查並套用。

### 4. 手動等價（腳本不可用時）

`PATCH /api/settings`，body：

```json
{ "blockedProviders": ["aihorde", "auggie", "chipotle", "cloudflare-playground", "devin-cli-agentic", "duckduckgo-web", "felo-web", "codex-app-server", "opencode", "theoldllm", "uncloseai", "veoaifree-web", "zcode"] }
```

認證用 loopback header `x-omniroute-cli-token`（見 OmniRoute `src/lib/machineToken.ts`），或 dashboard session。`blockedProviders` 不是 security-impacting key，不必管理密碼。先 GET settings，用 `If-Match: <settingsRevision>` 避免衝突。

## Rules and Limitations

- 相對路徑以本 Skill 資料夾為準。
- 不要用 `omniroute providers remove` 或直接 DELETE `provider_connections`；免鑑權本來就沒有連線列。
- 不要把 `codex-app-server` 跟已設定的 OAuth `codex` 搞混；本流程不刪付費／OAuth 連線。
- 不要改 `noAuthFallbackDisabledProviders` 來關真正的 no-auth 提供者。
- 不要把 CLI token、API key、管理密碼寫進 skill、log 或回覆。
- UI 補丁不是官方設定；`npm`／source 升級會蓋掉 chunk，必須重新套用對應補丁並以 HTTP 實際回應驗證。
- `Show configured only` 是 Dashboard 顯示層篩選，不是 `/v1/models` catalog 的刪除或安全限制；直接指定完整 model ID 仍可能路由。
- 套用 configured-only 補丁前必須備份 chunk，且只接受唯一匹配；找不到或多重匹配時停止，不要盲目替換壓縮檔。
- 不要為了藏畫面去改 OmniRoute 核心路由或清 catalog 原始碼，除非使用者明確要求改上游。

## Pitfalls

- Dashboard「已設定」模式仍會顯示被封鎖的 no-auth（官方當成 configured）。
- `keys reveal` 對本機管理金鑰可能 404；管理 API 用 CLI token 即可。
- Windows 上 Python `subprocess` 呼叫 `omniroute` 會找不到檔案；要用 `omniroute.cmd`（腳本已處理）。
- Windows 上 `node -e` 寫檔時不要用 Git Bash 的 `/tmp`，Node 會變成 `C:\tmp\...`。
- 壓縮 chunk 的變數名（`eC` / `eG` / `eH`）隨 build 而變；`hide_noauth_ui.py` 對不到條件時不要亂替換，改搜 `noAuthEntriesAll.length > 0 || blockedNoAuthEntries.length > 0` 或新的 minified 字串。
- 套 UI 補丁卻沒先 `block`：可見卡片仍在，整區還是會出現。
- OpenCode Free（`oc/`）一併封鎖後，zero-config `auto` 會少掉免費後備。

## Verification

```powershell
python D:\OB\skills\omniroute-noauth-providers\scripts\block_noauth.py status
omniroute --quiet --no-color providers list --json
omniroute --quiet --no-color models --search oc
python D:\OB\skills\omniroute-noauth-providers\scripts\hide_noauth_ui.py status
```

通過條件：

1. `status` 顯示全部 catalog no-auth id 都在 `blockedProviders`。
2. `providers list` 仍只有原本的付費／OAuth 連線。
3. `models --search oc` / `ddgw` / `felo` 為 No models found。
4. 若有套 UI 補丁：`hide_noauth_ui.py status` 為 patched；硬重新整理後頁面沒有「免鑑權提供者」標題與 Disabled 卡片。
5. YAML frontmatter `name` 與資料夾 `omniroute-noauth-providers` 一致。
