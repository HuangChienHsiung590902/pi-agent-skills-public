---
name: pi-coding-agent-setup
description: >-
  Set up and manage the local `pi` coding agent (`@earendil-works/pi-coding-agent`, npm global install at `C:\Users\HCH\AppData\Roaming\npm\pi`, config under `~/.pi/agent/`) — its OmniRoute integration (`npm:omniroute-pi-ext-integration` in `~/.pi/agent/settings.json`'s `packages` array, enriches the Ctrl+P model picker with OmniRoute provider/model metadata), the custom multi-line footer/HUD extension (`~/.pi/agent/extensions/hud.ts`, an `ExtensionAPI` script showing session/git info, model/provider/thinking level, a context-usage bar, subscription-oriented cache diagnostics, OmniRoute actual routing/latency/health, quota, and turn/tool counters — distinct from the Claude Code `omniroute-hud` statusLine skill), and how pi shares the same `~/.claude/skills` directory as Claude Code (`"skills": ["~/.claude/skills"]` in pi's settings, so skills here are visible to both agents). Use when the user mentions the `pi` CLI/coding agent, pi 的 HUD/footer/狀態列, pi extensions, `omniroute-pi-ext-integration`, or reconfiguring pi's model/provider defaults, packages, or skills path.
---

# pi Coding Agent — 本機安裝與設定

## 這個 skill 涵蓋什麼

`pi`（`@earendil-works/pi-coding-agent`）是本機另一套透過 npm 全域安裝的 coding agent CLI，跟 Claude Code 是**兩支獨立的程式**，但**共用同一個 `~/.claude/skills` skill 庫**（見下方「skills 共用」一節）。這份 skill 記錄：

1. pi 本體的安裝位置與設定檔結構
2. OmniRoute 整合套件（`omniroute-pi-ext-integration`）怎麼接上
3. 自訂 HUD/footer 擴充套件（`hud.ts`）的功能與安裝方式
4. 為什麼 pi 和 Claude Code 的 skill 要放在一起管理

## 安裝位置

- **執行檔**：npm 全域安裝，`C:\Users\HCH\AppData\Roaming\npm\pi`（Git Bash 用）/ `pi.cmd`（cmd/PowerShell 用）
- **套件版本**：`@earendil-works/pi-coding-agent@0.83.0`（用 `npm ls -g --depth=0` 確認目前版本）
- **設定根目錄**：`~/.pi/agent/`
  - `settings.json`：主設定（預設 provider/model/thinking level、`packages`、`skills` 路徑）
  - `auth.json`：各 provider 的 OAuth/API token（機密，不要印出全文）
  - `models.json` / `models-store.json`：已同步的模型清單與 metadata（含 context window、reasoning、vision 等資訊，由 `omniroute-pi-ext-integration` 同步進來）
  - `extensions/`：本機自訂的 `ExtensionAPI` 擴充腳本（`.ts`），例如 `hud.ts`、`herdr-panel-command.ts`、`webmcp-command.ts` 與 `connect-chrome-command.ts`
  - `npm/`：pi 自己管理的一個小型 npm 專案（`package.json` + `node_modules/`），用來安裝 `packages` 陣列裡宣告的 npm 套件（例如 `omniroute-pi-ext-integration`），不是本機專案的 node_modules
  - `sessions/`：對話 session 紀錄（`.jsonl`），環境變數 `PI_SESSION_FILE` 指向目前這個 session 的檔案

### 目前的 `~/.pi/agent/settings.json`

```json
{
  "lastChangelogVersion": "0.83.0",
  "theme": "dark",
  "defaultProvider": "omni",
  "defaultModel": "qwythos/qwythos-9b-v2-awq",
  "defaultThinkingLevel": "medium",
  "reserveTokens": 8192,
  "packages": [
    "npm:omniroute-pi-ext-integration",
    "npm:pi-playwright",
    "npm:pi-mcp-adapter",
    "npm:pi-hermes-memory"
  ],
  "skills": [
    "D:/.system/.claude/skills"
  ]
}
```

- `defaultProvider: "omni"` + `defaultModel: "qwythos/qwythos-9b-v2-awq"`：預設走 OmniRoute 閘道器接遠端 vLLM qwythos。`reserveTokens: 8192` 是為了避免 pi 預設保留 16384 output tokens 撞 vLLM context 上限。
- `packages`：pi 啟動時會確保這裡列的 npm 套件已安裝在 `~/.pi/agent/npm/node_modules/` 下，並依套件 `package.json` 裡的 `"pi": { "extensions": [...] }` 欄位自動載入對應的擴充功能。
- `skills`：**pi 讀取 skill 的路徑，指向 Claude Code 的 `~/.claude/skills`**，兩邊看到的 skill 清單是同一份。

## OmniRoute 整合套件：`omniroute-pi-ext-integration`

- npm 套件名稱：`omniroute-pi-ext-integration`（作者 md-riaz，MIT）
- 用途：讓 pi 的 `Ctrl+P` 模型選擇器可以瀏覽 OmniRoute 的 provider/model 組合，並同步豐富化的 metadata（context window、max tokens、reasoning 能力、vision 能力）進 `~/.pi/agent/models.json` / `models-store.json`。
- 安裝方式：不是 `npm install` 到全域，而是寫進 `~/.pi/agent/settings.json` 的 `packages` 陣列（`"npm:omniroute-pi-ext-integration"`），pi 啟動時自己解析安裝到 `~/.pi/agent/npm/node_modules/omniroute-pi-ext-integration/`，實際安裝清單記錄在 `~/.pi/agent/npm/package.json` + `package-lock.json`（一個獨立的、pi 自己管理的小 npm 專案，peer dependency 是 `@earendil-works/pi-coding-agent >=0.60.0`）。
- 若要升級/重裝：改 `packages` 裡的版本號（例：`"npm:omniroute-pi-ext-integration@^2.0.1"`），重啟 pi 讓它重新解析；或直接進 `~/.pi/agent/npm/` 手動 `npm install`。
- 若要移除：從 `packages` 陣列刪掉這一行，重啟 pi。


### Qwythos / vLLM 模型快取同步（重要）

pi 的 `reserveTokens` 只控制壓縮/保留額度；實際送到 OpenAI-compatible 後端的
`max_tokens` 還會受 `~/.pi/agent/models.json` 裡的 model metadata 影響。

目前 qwythos 兩個 model id 都必須是：

```json
"contextWindow": 98304,
"maxTokens": 8192
```

要檢查/修正：

```bash
python - <<'PY'
import json, pathlib
p = pathlib.Path(r'D:/.system/.pi/agent/models.json')
data = json.loads(p.read_text(encoding='utf-8'))
ids = {
  'qwythos/qwythos-9b-v2-awq',
  'openai-compatible-chat-dfdac666-5d49-4585-a3d4-210e5f8613e0/qwythos-9b-v2-awq',
}
for provider in data.get('providers', {}).values():
    for m in provider.get('models', []):
        if m.get('id') in ids:
            m['contextWindow'] = 98304
            m['maxTokens'] = 8192
p.write_text(json.dumps(data, ensure_ascii=False, indent=2) + '
', encoding='utf-8')
PY
```

如果只改 `settings.json` 的 `reserveTokens`，但漏改 `models.json`，pi 仍可能送
`max_tokens: 16384`，造成遠端 vLLM 報：

```text
input 81921 + output 16384 = 98305 > max_model_len 98304
```

改完後重開 pi session；若模型清單被 `/omni sync` 覆蓋，再回頭檢查 OmniRoute
端 provider/model metadata。

## 自訂 HUD / Footer 擴充套件：`hud.ts`

**位置**：`~/.pi/agent/extensions/hud.ts`（本機手寫的 TypeScript，不是 npm 套件，pi 啟動時會自動掃描 `~/.pi/agent/extensions/` 底下的 `.ts` 檔並載入）

### 功能

用 `ExtensionAPI`（`@earendil-works/pi-coding-agent` 提供）在 pi 的 TUI 畫面掛一個多行 footer，即時顯示：

1. **第一行**：目前工作目錄名稱、session 名稱/ID、git branch
2. **第二行**：目前 model id、provider、thinking level。
3. **第三行**：context window 使用率長條圖（`[####----]` 形式）+ token 數 / window 上限 + 百分比。
4. **第四行**：其他 extension/service 狀態 chips。
5. **第五行**：最近一筆 OmniRoute 實際路由（requested model → provider/model）、帳號、延遲、HTTP status、request type、combo、格式轉換與 connection ID。
6. **第六行**：OmniRoute 近 24 小時請求總數、成功率、錯誤數與平均延遲。
7. **第七行**：OmniRoute 可用供應商清單；最近被調用的 provider 會變色並標示 `*`。
8. **第八行**：OmniRoute 各 provider 近 24 小時流量、成功率與平均延遲。
9. **第九行**：OmniRoute 各 model 近 24 小時調用量與錯誤數。
10. **第十行**：OmniRoute 各連線額度剩餘比例與重置倒數。
11. **第十一行**：本 session 觀察到的 Skills。

### 運作機制（供之後修改參考）

- 掛在 `pi.on("session_start", ...)`：session 開始時用 `ctx.ui.setFooter(...)` 註冊一個 render function，`footerData.onBranchChange` 訂閱 git branch 變化以觸發重繪；OmniRoute 路由狀態與額度每 30 秒唯讀輪詢一次。
- OmniRoute 資訊來源：唯讀查詢 OmniRoute `storage.sqlite` 的 `call_logs`、`provider_connections` 與 `quota_snapshots`；不會修改資料庫。
- `turn_start` / `turn_end` / `tool_call` / `agent_end` 這幾個生命週期事件用來維持工作狀態與 Skills 觀察；顯示哪些 HUD 區塊由 `/hud` 設定控制。
- Context 使用率：`ctx.getContextUsage()` 搭配 `ctx.model.contextWindow` 算百分比，並用 `renderBar()`（本檔案內的 helper）畫成文字長條圖。
- 顏色：透過 `theme.fg("dim"|"accent"|"muted"|"success", text)` 取色，不是寫死 ANSI code。

### 修改/重裝方式

HUD 欄位現在可在 pi 內用 `/hud` 互動式問答逐項增刪，設定保存於 `D:/.system/.pi/agent/hud.json`；`/hud reset` 可恢復預設。只有要修改 HUD 的資料來源或排版邏輯時，才直接編輯 `~/.pi/agent/extensions/hud.ts`（純文字 TypeScript，pi 用 `node --experimental-strip-types` 之類的機制直接執行，不需要額外編譯步驟），並重開新的 pi session。

## 跟 Claude Code「HUD」的差異（容易搞混，務必分清楚）

| | pi 的 `hud.ts` | Claude Code 的 `omniroute-hud` skill |
|---|---|---|
| 語言/型別 | TypeScript，`ExtensionAPI` | Python 腳本 |
| 掛載位置 | `~/.pi/agent/extensions/hud.ts` | `~/.claude/settings.json` 的 `statusLine.command` |
| 顯示內容 | pi session 本身的 token/turn/context 統計 | OmniRoute 閘道器存活狀態、目前 provider/model、多帳號額度 |
| 資料來源 | pi 內部的 `ctx.sessionManager` / `ctx.getContextUsage()` | 直接讀 OmniRoute 的 `~/.omniroute/storage.sqlite`（唯讀）+ TCP 存活檢查 |
| 適用對象 | 只有 pi 看得到（pi 的 TUI footer） | 只有 Claude Code 看得到（Claude Code 的狀態列） |

兩者是**完全獨立的兩套顯示機制**，跑在不同的 agent 裡，不要互相套用設定或程式碼。OmniRoute 額度/存活狀態的細節見 `omniroute-hud` skill；OmniRoute 本體的原生安裝見 `omniroute-native` skill。

## 為什麼這個 skill 放在 `~/.claude/skills/`，跟 Claude Code 相關 skill 放一起

`~/.pi/agent/settings.json` 的 `"skills": ["~/.claude/skills"]` 表示 pi **直接讀取 Claude Code 的 skill 目錄**，兩邊是同一份 skill 庫、同時生效，不是各自獨立維護兩份。因此：

- **不要**另外建一個 `~/.pi/skills/` 之類的獨立目錄放 pi 專屬的 skill，那樣 pi 讀得到但管理會分散成兩處。
- 所有跟本機 agent CLI（無論是 Claude Code 還是 pi）的安裝、設定、擴充套件相關的 skill，一律放在 `~/.claude/skills/` 底下，方便統一搜尋、統一備份、統一維護（可與 `claude-cli` skill 對照參考——那個記錄的是「用 Claude Code CLI 呼叫本機 Claude 執行任務」，這個 skill 記錄的是「pi CLI 本身的安裝設定」，兩者主題不同但都屬於「本機 coding agent CLI 管理」這個分類，因此放在同一個 skill 庫方便一起找）。

## 環境變數（pi session 內可用，用於判斷目前是否跑在 pi 底下）

```
PI_CODING_AGENT=true
PI_REASONING_LEVEL=<low|medium|high>
PI_SESSION_FILE=<絕對路徑到目前 session 的 .jsonl>
PI_PROVIDER=<目前 provider，例：omni>
PI_MODEL=<目前 model id，例：claude/claude-sonnet-5>
PI_SESSION_ID=<UUID>
```

若腳本/擴充功能需要判斷「目前是不是在 pi 裡執行」，檢查 `PI_CODING_AGENT` 是否為 `"true"` 即可，不需要用更複雜的偵測方式。

---

## Conformance Addendum

## When to Use
Set up and manage the local `pi` coding agent (`@earendil-works/pi-coding-agent`, npm global install at `C:\Users\HCH\AppData\Roaming\npm\pi`, config under `~/.pi/agent/`) — its OmniRoute integration (`npm:omniroute-pi-ext-integration` in `~/.pi/agent/settings.json`'s `packages` array, enriches the Ctrl+P model picker with OmniRoute provider/model metadata), the custom multi-line footer/HUD extension (`~/.pi/agent/extensions/hud.ts`, an `ExtensionAPI` script showing session/git info, model/provider/thinking level, a context-usage bar, subscription-oriented cache diagnostics, OmniRoute actual routing/latency/health, quota, and turn/tool counters — distinct from the Claude Code `omniroute-hud` statusLine skill), and how pi shares the same `~/.claude/skills` directory as Claude Code (`"skills": ["~/.claude/skills"]` in pi's settings, so skills here are visible to both agents). Use when the user mentions the `pi` CLI/coding agent, pi 的 HUD/footer/狀態列, pi extensions, `omniroute-pi-ext-integration`, or reconfiguring pi's model/provider defaults, packages, or skills path.

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
