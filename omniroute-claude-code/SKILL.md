---
name: omniroute-claude-code
version: 1.1.0
description: >-
  在 Windows 上把 Claude Code 接到本機 OmniRoute，讓 Claude Code 透過 gateway
  model discovery 讀取 OmniRoute 的可用模型。當使用者提到 Claude Code 連 OmniRoute、
  /model 看不到 OmniRoute 模型、要使用 Qwen/GPT/GLM/DeepSeek、要驗證
  /v1/models、設定 discovery aliases、建立 Profile 或排查 401 時使用。
---

# OmniRoute + Claude Code：Gateway Model Discovery

## 核心結論（先讀這裡）

Claude Code **可以**從 LLM gateway 的 `/v1/models` 動態填入 `/model` picker；不是只能顯示內建 Claude 模型，也不是把 `ANTHROPIC_BASE_URL` 指向 OmniRoute 就會自動發生。

要讓 Claude Code 讀取 OmniRoute 的 LLM，四件事必須同時成立：

1. `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1` 已設定。
2. `ANTHROPIC_BASE_URL` 是 OmniRoute 的 gateway root，例如 `http://localhost:20128`，不要在這裡加 `/v1`。
3. Claude Code 使用的 `ANTHROPIC_AUTH_TOKEN` 是 OmniRoute **目前有效的 gateway API key**。
4. OmniRoute 的 authenticated `GET /v1/models` 能回傳模型清單；非 Claude 模型若要以 Claude Code discovery alias 顯示，還要開啟 `EXPOSE_CC_DISCOVERY_ALIASES=1` 或等效 feature flag。

缺少第 3 或第 4 項時，單看 discovery 環境變數會造成誤判。最常見的實際故障是 `/v1/models` 回 `401`，因此 `/model` 沒有 gateway 模型。

## When to Use

- 將 Claude Code 指向本機或遠端 OmniRoute。
- Claude Code 已能執行 request，但 `/model` 看不到 OmniRoute 的 Qwen、GPT、GLM、DeepSeek 或其他模型。
- 使用者要讓 Claude Code 讀取 OmniRoute 的 live model catalog。
- 排查 `401 Authentication required`、`401 Invalid API key`、model discovery 沒有結果。
- 建立、修復或檢查 `~/.claude/profiles/<name>/settings.json`。
- 使用 `omniroute run claude --model` 精準指定模型。

本 Skill 只處理 Claude Code ↔ OmniRoute 接線與 discovery；OmniRoute 安裝、服務啟停、provider OAuth 與一般資料庫診斷另參考 `omniroute-native`。

## Inputs and Outputs

**輸入**

- OmniRoute base URL，預設 `http://localhost:20128`。
- 使用者在本機安全取得的 OmniRoute API key；不要要求使用者把完整 key 貼到對話。
- 目標模型 ID，例如 `claude/cx/gpt-5.6-luna-high` 或 `claude/qct/qwen3.8-max`。
- 使用者要的是 live `/model` picker、單次指定模型，還是 Profile。

**輸出**

- 可重現的 launcher 命令或 Claude Code Profile。
- authenticated `/v1/models`、alias 數量與目標模型存在性的驗證結果。
- 清楚區分「catalog 已回傳」、「`/model` picker 顯示」與「request 實際使用的 model」。
- 不在 Skill、settings、log、測試檔或回覆中暴露完整 credential。

## URL 與 protocol 規則

### Claude Code 的 Anthropic base URL

Claude Code 使用 Anthropic Messages API，設定 gateway root：

```text
ANTHROPIC_BASE_URL=http://localhost:20128
```

Claude Code 會在此 root 下使用 `/v1/messages`，gateway discovery 使用 `/v1/models`。

不要把 Claude Code 的 `ANTHROPIC_BASE_URL` 設成：

```text
http://localhost:20128/v1
```

`/v1` 是 OpenAI-compatible client 常用的 base URL；對 Claude Code 的 Anthropic base URL 會造成路徑重複或 protocol 不匹配。

### OmniRoute 的 discovery aliases

Claude Code 的 gateway discovery 會讀取 OmniRoute 回傳的 model IDs。OmniRoute 可將非 Claude 模型鏡像成 Claude Code 可接受的 alias：

```text
claude/<provider>/<model>
```

例如：

```text
claude/cx/gpt-5.6-luna-high
claude/qct/qwen3.8-max
claude/qwen-cloud-token-plan/deepseek-v4-pro
```

OmniRoute 設定：

```env
EXPOSE_CC_DISCOVERY_ALIASES=1
```

也可在 Dashboard 的 `Settings → Feature Flags → Claude Code Discovery Aliases` 開啟。環境檔或 feature flag 改動後，依 `omniroute-native` 的安全流程重啟服務；不要只殺掉任意 port PID。

## 正確診斷順序

### 1. 先驗證服務存活

```powershell
omniroute health
claude --version
```

`health` 只證明服務活著，**不證明 Claude Code 已通過 gateway 認證**。

### 2. 直接驗證 authenticated `/v1/models`

不要先改 Claude Code 設定，也不要先猜模型 ID。先在目前 PowerShell process 安全取得 key：

```powershell
$env:OMNIROUTE_API_KEY = Read-Host "OmniRoute API key"
```

呼叫 catalog：

```powershell
$models = Invoke-RestMethod `
  -Uri "http://127.0.0.1:20128/v1/models" `
  -Headers @{ Authorization = "Bearer $env:OMNIROUTE_API_KEY" }

$models.data | Where-Object { $_.id -like "claude/*" } | Select-Object -ExpandProperty id
```

檢查目標模型：

```powershell
$models.data.id -contains "claude/cx/gpt-5.6-luna-high"
```

結果判讀：

- `200` + `data`：OmniRoute catalog 可供 Claude Code discovery 使用。
- `401 Authentication required`：沒有送有效 Bearer token；不要只重設 discovery flag。
- `401 Invalid API key`：token 已送出，但不是目前有效的 OmniRoute gateway key；重新從本機 Dashboard/API Keys 或安全的 launcher context 取得。
- `200` 但沒有 `claude/...`：檢查 `EXPOSE_CC_DISCOVERY_ALIASES`、feature flag、provider 是否有 active credential，以及 catalog cache。
- `200` 且有 alias，但 `/model` 沒有：檢查 Claude Code 的 `availableModels`、`modelPicker`、`replaceBuiltInOptions`、版本與是否真的完整重啟。

不要用未驗證的 `omniroute models list` 結果取代 `/v1/models` 測試；Claude Code 需要的是 HTTP endpoint 的實際回應。

### 3. 查目前 Claude Code 的設定

應至少有：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:20128",
    "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY": "1"
  }
}
```

`ANTHROPIC_AUTH_TOKEN` 可以由 `omniroute launch` / `omniroute run` 在 process 啟動時注入；不要把完整 token 寫進可提交的 Skill 或專案設定。

若有企業或本機限制，檢查：

- `availableModels`：若存在，discovered model 必須在 allowlist 中。
- `modelPicker`：可自訂 picker；`replaceBuiltInOptions: true` 會隱藏內建 lineup、gateway-discovered models 與 custom option，只顯示明確列出的 rows。
- `modelPicker.options[].behavesAs`：若 Claude Code 尚不認識 gateway model，可用已知 Claude model 套用 client-side handling，但不會改變送出的 model ID。

## 啟動方式

### 最推薦：`omniroute run claude`

這個命令適合精準指定 model，並由 OmniRoute 注入 gateway URL、token、discovery flag：

```powershell
$env:OMNIROUTE_API_KEY = Read-Host "OmniRoute API key"
omniroute run claude `
  --api-key-env OMNIROUTE_API_KEY `
  --model "claude/cx/gpt-5.6-luna-high"
```

Qwen 範例：

```powershell
omniroute run claude `
  --api-key-env OMNIROUTE_API_KEY `
  --model "claude/qct/qwen3.8-max"
```

單行版：

```powershell
omniroute run claude --api-key-env OMNIROUTE_API_KEY --model "claude/cx/gpt-5.6-luna-high"
```

### `omniroute launch`

`launch` 適合啟動 Claude Code、注入 gateway 與 token、或使用 Profile：

```powershell
$env:OMNIROUTE_API_KEY = Read-Host "OmniRoute API key"
omniroute launch --api-key $env:OMNIROUTE_API_KEY
```

如果要精準指定任意 model，優先使用 `run claude --model`；不要假設每個 `launch` 版本都接受或處理任意 model flag。

### 直接執行 `claude`

PowerShell：

```powershell
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:20128"
$env:ANTHROPIC_AUTH_TOKEN = Read-Host "輸入 OmniRoute API token"
$env:CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY = "1"
$env:ANTHROPIC_MODEL = "claude/cx/gpt-5.6-luna-high"
claude --dangerously-skip-permissions
```

CMD：

```cmd
set "ANTHROPIC_BASE_URL=http://127.0.0.1:20128"
set "ANTHROPIC_AUTH_TOKEN=輸入token"
set "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1"
set "ANTHROPIC_MODEL=claude/cx/gpt-5.6-luna-high"
claude --dangerously-skip-permissions
```

不要在 CMD 貼 PowerShell 的 `$env:` 語法；那只會造成環境變數沒有真正設定。

## `/model`、direct `--model` 與 Profile 的區別

- **Gateway discovery**：Claude Code 從 authenticated `/v1/models` 取得模型，符合條件的模型可進入 `/model` picker。
- **`/model` picker**：會合併內建模型、gateway discovery、custom/modelPicker 設定；allowlist 或 `replaceBuiltInOptions` 可能過濾結果。
- **`--model` / `ANTHROPIC_MODEL`**：直接指定 request 使用的 model，不代表該 model 一定顯示在 picker。
- **Profile**：適合固定一個 model 或一組 gateway 設定，透過 `CLAUDE_CONFIG_DIR` / `omniroute launch --profile` 啟動。

因此不能把「catalog 有模型」與「picker 一定顯示」混為一談；但也不能再宣稱 `/model` 永遠只會顯示內建 Claude 模型。正確做法是先驗證 `/v1/models`，再檢查 picker 過濾設定。

## Profile

先 dry-run：

```powershell
$env:OMNIROUTE_API_KEY = Read-Host "OmniRoute API key"
omniroute setup-claude `
  --api-key-env OMNIROUTE_API_KEY `
  --only qwen `
  --dry-run
```

若目前版本不接受 `--api-key-env`，不要把 key 寫入檔案；改在 process env 設定 `OMNIROUTE_API_KEY` 後執行：

```powershell
omniroute setup-claude --only qwen --dry-run
```

確認 catalog 後再寫 profile：

```powershell
omniroute setup-claude --only qwen
```

啟動：

```powershell
omniroute launch --profile <profile-name>
```

Profile 的 `settings.json` 可保存非秘密設定：

```json
{
  "model": "claude/qct/qwen3.8-max",
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:20128",
    "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY": "1"
  }
}
```

不要把 raw API key 寫入 Profile。

## Troubleshooting

### `/model` 沒有 OmniRoute 模型

按順序檢查：

1. 完全退出 Claude Code，再重新啟動；discovery 不應假設在既有 process 內即時重載。
2. 實測 authenticated `GET /v1/models` 是否 200。
3. 確認 Claude Code process 使用的 token 與 OmniRoute active key 相同；只比較遮罩後的前後幾碼，不要印出完整 key。
4. 確認 OmniRoute 有 `EXPOSE_CC_DISCOVERY_ALIASES=1`，或 Dashboard feature flag 已啟用。
5. 檢查 `availableModels`、`modelPicker`、`replaceBuiltInOptions` 是否過濾 discovery。
6. 確認沒有使用錯誤的 `ANTHROPIC_BASE_URL=http://localhost:20128/v1`。
7. 確認 Claude Code 版本支援 gateway discovery；目前本機已驗證的版本是 `2.1.280`。

### `Not logged in` / `/login`

先分開測試 gateway 存活與 gateway auth：

```powershell
omniroute health
curl.exe -i http://127.0.0.1:20128/v1/models
```

health 正常只代表 OmniRoute 存活；`/v1/models` 的 401 才表示目前 process 沒送有效 OmniRoute token。使用 launcher 或在目前 shell 用 `Read-Host` 設 token，然後重新啟動 Claude Code。

### `isn't described by this version's model catalog`

使用完整 discovery alias：

```text
錯誤：cx/gpt-5.6-luna-high
正確：claude/cx/gpt-5.6-luna-high
```

若本機 catalog enforcement 仍阻擋未知 alias，在已確認上游 context 後可暫時使用：

```env
CLAUDE_CODE_DISABLE_UNKNOWN_MODEL_WINDOW_ENFORCEMENT=1
```

這只停用本機未知模型視窗的阻擋，不會替上游模型增加 context，也不會修正錯誤 model ID。

### `.env` 改了但 401 仍存在

不要只相信 `.env` 內容。既有 supervisor、背景 OmniRoute process、資料庫 feature flag 或啟動時環境可能仍然有效。以實際 HTTP `/v1/models` 結果為準，依 `omniroute-native` 的停機/啟動流程完整重啟後再測。

### 更新失效的 gateway API key

如果 `/v1/models` 回 `401`，先不要修改 model ID 或重設 discovery flag。確認目前 OmniRoute server 的 API key 清單；若沒有有效 key，透過 Dashboard 或受保護的管理 API 建立一把專用 key，然後只在本機安全流程中更新 Claude Code 的 `ANTHROPIC_AUTH_TOKEN`。不要把完整 key 貼到聊天、Skill、log 或可提交檔案。

API 建立請求的 payload 使用 `name` 欄位，例如：

```json
{"name":"Claude Code OmniRoute"}
```

建立後只保留在本機設定或啟動 process，並立即以同一 token 驗證 `GET /v1/models`。若要更新 `settings.json`，保留非秘密設定：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:20128",
    "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY": "1"
  }
}
```

實際 token 不應寫入 Skill 或回覆；Claude Code 完全退出再重開後才會重新載入 discovery。

## Rules and Limitations

- 不要把完整 API key 寫入 Skill、Claude settings、Profile、log、測試檔或回覆。
- 不要把 `http://localhost:20128/v1` 當成 Claude Code 的 `ANTHROPIC_BASE_URL`。
- 不要宣稱 discovery flag 單獨就足夠；一定要驗證 authenticated `/v1/models`。
- 不要宣稱 `/model` 永遠只有內建模型；gateway discovery 開啟且未被 picker/allowlist 過濾時，Claude Code 可以顯示 gateway models。
- 也不要宣稱 OmniRoute catalog 的每一個理論模型都一定可用；實際清單受 active credentials、quota、provider health、region 與 model sync 影響。
- 不要使用未經目前 `/v1/models` 驗證的舊 model ID。
- `claude/...` alias 是 OmniRoute 提供給 Claude Code discovery/Anthropic lane 的鏡像 ID；不要自行刪掉 prefix。
- 修改 OmniRoute 服務設定或重啟前，說明可能中斷現有 request，並遵守 `omniroute-native` 的安全停機流程。
- 測試 helper 不得內嵌 API key；使用 `OMNIROUTE_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN` 環境變數。

## Reference

- `references/omniroute-gateway-discovery.md` — Claude Code gateway discovery、OmniRoute token、aliases、MCP 關係與重灌後恢復流程的完整參考。

## Verification

完成設定後至少執行：

```powershell
omniroute health
claude --version
```

再使用目前 process 的 key 驗證：

```powershell
$models = Invoke-RestMethod `
  -Uri "http://127.0.0.1:20128/v1/models" `
  -Headers @{ Authorization = "Bearer $env:OMNIROUTE_API_KEY" }
$models.data.id -contains "claude/cx/gpt-5.6-luna-high"
```

如果要驗證 picker，完全重啟 Claude Code 後執行：

```text
/model
```

回報時分別說明：

1. `/v1/models` 是否 HTTP 200，以及 catalog/alias 數量。
2. 目標 model ID 是否存在。
3. `/model` 是否因 picker 設定被過濾。
4. 實際 request 使用的是哪個 model（以 OmniRoute call log 或 debug evidence 為準）。
5. 仍需使用者手動完成的 token、登入或重啟步驟。

## Verified local evidence (2026-09-23)

本機 OmniRoute `3.8.50` 的 authenticated `/v1/models` 本回合驗證回傳 HTTP 200、879 個模型，其中 335 個是 `claude/...` aliases，涵蓋 28 個 alias provider；目前設定的 `claude/cx/gpt-5.6-luna` 也存在於 catalog。這是當時的 live catalog，不是永久保證；之後應每次重新查 endpoint，而不是硬編號量。

本次錯誤根因是 Claude Code 使用的 `ANTHROPIC_AUTH_TOKEN` 與 OmniRoute active gateway key 不一致，造成 `/v1/models` 401；不是 discovery flag 不支援，也不是 OmniRoute 沒有模型。更新 gateway key 並將 `EXPOSE_CC_DISCOVERY_ALIASES=1` 寫入使用者資料目錄 `.env`、依安全流程重啟 OmniRoute 後，驗證恢復為 HTTP 200。後續診斷必須先驗證 token 與 endpoint，不可只看 settings.json 裡是否有 flag。
