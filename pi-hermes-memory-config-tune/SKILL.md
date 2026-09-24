---
name: pi-hermes-memory-config-tune
description: "修 pi-hermes-memory 的自動 review / 整併出錯（特別是 No API provider registered for api omni-prompt-tools）與調整 MEMORY/USER/PROJECT 字元上限。用在：看到 Memory auto-review failed in both transports、direct 與 subprocess 同時失敗、omni-prompt-tools provider 未註冊、記憶上限 5000 字元不夠用要調大、或要換 llmModelOverride 時。"
---

# pi-hermes-memory 自動 review 修復與記憶上限調整

## When to Use
- 使用者貼出 `Warning: Memory auto-review failed in both transports. Direct: provider_error: No API provider registered for api: omni-prompt-tools. Subprocess: ...`
- pi-hermes-memory 的背景 review / consolidation / flush 持續失敗
- 使用者覺得 memory 容量（預設 5000 字元）不夠用，想調到 10000 或更大
- 要為 memory 自動 review 指定獨立 model（`llmModelOverride`）或 transport

搭配 `pi-hermes-memory-usage`（運作機制與安裝檢查）一起看；本 skill 只處理「設定修復與調參」。

## 背景：為什麼會出 "No API provider registered for api: omni-prompt-tools"
- 本機 pi 的 omni provider 由 `omniroute-pi-ext-integration` extension 在執行期呼叫 `pi.registerProvider()` 註冊，`models.json` 裡 omni 的 model 全部標記 `api: omni-prompt-tools`（這個 api 名稱不是 pi 內建的，只有 extension 載入時才存在）。
- pi-hermes-memory 的 **direct transport** 在 Pi 主行程內直接呼叫 `completeSimple()`；某些情境下主行程的 api provider registry 沒有拿到 extension 註冊的 handler，就報 "No API provider registered"。
- pi-hermes-memory 的 **subprocess transport** 開 `pi -p --no-extensions -e <hermes 自身>` 子行程；`--no-extensions` 會把 settings.json 的 packages（含 omniroute extension）全關掉，所以子行程裡 `omni-prompt-tools` 一樣不存在 → 兩邊都失敗。

## Procedure

### 1. 讀現行設定
設定檔：`D:\.system\.pi\agent\hermes-memory-config.json`（`~/.pi` 是指向 `D:\.system\.pi` 的 symlink，一律用 D 槽真實路徑）。
先備份：
```bash
cp D:\.system\.pi\agent\hermes-memory-config.json D:\.system\.pi\agent\hermes-memory-config.json.bak-$(date +%Y%m%d-%H%M%S)
```

### 2. 找一個「子行程下真的可用」的 omni model
subprocess 會用 `--no-extensions` 只載入 hermes + `childExtensionPaths` 列出的 extension，所以 override model 必須在載入 omniroute extension 的條件下可用。用下列格式實測（每個 model 都實際打一次）：
```bash
pi -p --no-session --no-extensions \
  -e D:/.system/.pi/agent/npm/node_modules/omniroute-pi-ext-integration/index.ts \
  --model omni/<model-id> --thinking off "只回答 ok"
```
- 2026-08-28 實測：`omni/codex-auto-review` 回 `ok` ✅；`omni/oc/deepseek-v4-flash-free` 回 400 Model is unavailable；`omni/auto/*` 回 "Stream ended without finish_reason"（對 memory ops 的 JSON 解析不可靠）。
- 若 `omni/codex-auto-review` 之後壞了，重新掃 `pi --list-models --offline | grep omni` 找候選（`codex-*` 系列較穩）。

### 3. 寫入修正後的設定
```json
{
  "llmModelOverride": "omni/codex-auto-review",
  "llmThinkingOverride": "off",
  "reviewEnabled": true,
  "reviewTransport": "subprocess",
  "childExtensionPaths": [
    "D:/.system/.pi/agent/npm/node_modules/omniroute-pi-ext-integration/index.ts"
  ],
  "memoryCharLimit": 10000,
  "userCharLimit": 10000,
  "projectCharLimit": 10000
}
```
關鍵三件：
1. `reviewTransport: "subprocess"` — 避開 direct transport 未註冊 `omni-prompt-tools` 的問題。
2. `childExtensionPaths` 明確列出 omniroute extension 路徑 — hermes 子行程用 `--no-extensions` 後只會載入 hermes 自己 + 這個清單，omni provider 才會存在。
3. `llmModelOverride` 指向實測可用的 model，`llmThinkingOverride: "off"` 加快 review。

### 4. 調記憶上限（5000 → 10000 或更大）
`memoryCharLimit` / `userCharLimit` / `projectCharLimit` 三個分開控 MEMORY.md / USER.md / 專案記憶，**要調就三個一起調**，只調一個會遇到另一個先卡住。可用 `scripts/update_hermes_memory_config.py` 一鍵改（自動備份 + JSON 驗證）。

### 5. 驗證
```bash
node -e "JSON.parse(require('fs').readFileSync('D:/.system/.pi/agent/hermes-memory-config.json','utf8')); console.log('json ok')"
```
再重跑第 2 步的等效子行程命令確認 LLM 可回 `ok`。

### 6. 告知使用者
config 在 Pi 啟動時讀取，**目前 session 不會自動套用**，要開新 Pi session 後才生效。

## Pitfalls
- 不要只調 `memoryCharLimit` 忘記 `userCharLimit` / `projectCharLimit`，預設三個都是 5000。
- `llmModelOverride` 不能直接抄 `models.json` 的 model 就完事——很多 omni model 是 free 池會 400/418，必須實際呼叫驗證。
- `auto/*` 系列 model 會 "Stream ended without finish_reason"，對需要回傳結構化 JSON 的 memory ops 不可靠，避免選用。
- 改 `childExtensionPaths` 時路徑要用 Windows 絕對路徑且指向 `index.ts` 檔案本身，不是套件資料夾。
- `memoryOverflowStrategy` 若被設成 `reject`，寫滿上限會直接報錯而非自動 consolidation；調上限前先看這欄。
- 舊的 5000 字元說明在 `pi-hermes-memory-usage` skill 裡，那篇是套件預設值描述，與本 skill 的已調參現值（10000）不同，以設定檔為準。

## Verification
1. `hermes-memory-config.json` JSON parse 通過。
2. 第 2 步子行程命令回 `ok`。
3. 新開 Pi session 後觀察不再出現 "Memory auto-review failed in both transports"（可問 pi 用 memory_search 查剛寫入的測試條目）。
4. `D:\.system\.pi\agent\pi-hermes-memory\MEMORY.md` 寫入超過 5000 字元不再觸發上限報錯（上限已 10000）。

## Inputs and Outputs

### Inputs

- `D:/.system/.pi/agent/hermes-memory-config.json` 的目前設定。
- auto-review 錯誤、transport、`llmModelOverride` 與字元上限需求。

### Outputs

- 修改前的 timestamped backup。
- 更新後的 Hermes Memory 設定與實際測試結果。
- 若無法修復，清楚列出失敗的 provider、model 或 transport。

## Rules and Limitations

- 一律以 `D:/.system/.pi/agent` 為實際設定來源，不要只修改 `C:/Users/HCH` 的舊路徑。
- 不要在輸出、日誌或 Skill 內容中揭露 API key、OAuth token 或其他 credential。
- `llmModelOverride` 必須先確認 model 在目前 Pi/OmniRoute provider registry 中可用；不能只依模型名稱猜測。
- 修改設定後要重新啟動會讀取該設定的 Pi/Hermes process；不要把尚未重啟的檔案變更宣稱為已生效。
