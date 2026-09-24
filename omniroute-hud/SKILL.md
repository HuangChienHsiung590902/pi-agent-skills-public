---
name: omniroute-hud
description: Install a Claude Code status line that shows OmniRoute gateway liveness, the currently-active provider/model, remaining quota for EVERY registered provider connection across all providers — Claude, DeepSeek, etc. (session 5h / weekly 7d / credits_usd), broken connections, and phantom/ghost error counts. Reads OmniRoute's local SQLite DB (storage.sqlite) read-only plus a raw TCP liveness check on port 20128 — no HTTP auth needed. The fixed, ready-to-use script is bundled in this skill. Use when the user wants to see OmniRoute status or remaining model quota (including multi-account, multi-provider setups) directly in the Claude Code status line.
triggers:
  - omniroute hud
  - omniroute statusline
  - omniroute 額度
  - omniroute quota
  - 模型額度
  - quota statusline
  - omniroute status line
---

# OmniRoute HUD Status Line

在 Claude Code 的狀態列（`statusLine`）顯示 OmniRoute 閘道器（`http://localhost:20128`，此 session 的 `ANTHROPIC_BASE_URL` 實際後端）的即時狀態：存活與否、目前使用中的 provider/model、剩餘額度（5h / 7d / credits_usd）、失效連線數、幽靈節點錯誤數。輸出固定兩行，第一行是摘要，第二行是各帳號額度清單（避免單行過長）。

## 顯示範例

```
project | Sonnet 5 | ctx:42% | OmniRoute:up claude/claude-sonnet-5
hch590902:5h:23%(→2h10m) 7d:0%(→1d16h) hch.new*:5h:57%(→4h0m) 7d:59%(→4d3h) deepseek:main:credit:100%
```

- `ctx:NN%`：context window 使用率（綠 <70%、黃 <85%、紅 ≥85%）
- `OmniRoute:up`/`OmniRoute:down`：TCP 存活檢查（127.0.0.1:20128，0.3s timeout）
- `provider/model`：最近一筆 `call_logs` 紀錄（目前實際處理請求的那個 provider/model，並不代表就是唯一帳號）；5 分鐘內視為「使用中」顯示亮色（cyan），否則灰色（dim）
- **第二行：每個帳號各一組**，涵蓋 OmniRoute 裡登記的**所有 provider**（不限 Claude，DeepSeek 等其他 provider 一併列出），只要該帳號有 `quota_snapshots` 資料就會顯示。Claude 帳號名稱取 email `@` 前綴；非 Claude provider 加上前綴（例：`deepseek:main`）。目前實際在用的那個帳號會標示 `*` 並用亮色（cyan），其餘帳號用灰色（dim）——是全部帳號並列，不是只顯示一個。沒有 quota 快照的帳號不會顯示。
- `Nh:NN%` / `Nd:NN%`：對應 `quota_snapshots` 各 `window_key` 最新一筆 `remaining_percentage`（綠 ≥30%、黃 ≥10%、紅 <10% 或已耗盡），依帳號分開計算
- `(→距離重置時間)`：優先使用該額度視窗 `next_reset_at`；24 小時內顯示 `Xh Ym`（例：`1h5m`），超過 24 小時改用 `Xd Yh`（例：`2d3h`）。若 Claude `session (5h)` 的 `next_reset_at` 缺值，腳本會用最新快照 `created_at + 5h` 做 fallback 估算，避免 `5h` 只顯示百分比不顯示等待時間。
- `err:N`：`provider_connections` 中 `is_active=1` 且 `test_status` 為 `error`/`failed` 的連線數
- `ghost:N`：過去 24h 內 `call_logs` 有錯誤、但 `connection_id` 已找不到對應 `provider_connections` 的「幽靈節點」群組數（見 `omniroute-native` skill 的 phantom-node 說明）。通常是舊 provider connection 被刪除/重建後，舊 connection 的錯誤 log 還在 24h 視窗內；等最後一筆錯誤超過 24h 會自動消失。若要立刻消除，只能在確認後清掉對應 `call_logs`，不要盲刪整個 DB。

若 OmniRoute 存活但 DB 讀取失敗，顯示 `OmniRoute:up (db:<error>)`（灰階提示，不中斷狀態列，且不會有第二行）。若沒有任何帳號有 quota 快照，也不會印出第二行。

## 前置條件

- OmniRoute 已在本機以 native 方式安裝並跑在連接埠 20128（見 `omniroute-native` skill）。
- Windows 版 Python 3（此機器上驗證可用路徑：`C:\Users\HCH\AppData\Local\Programs\Python\Python312\python.exe`；Git Bash 的 `python3` 在此機常是 Store 別名 stub，不可用，須以 `where python`/`Get-Command python` 找真正路徑）。
- 只讀存取 `~/.omniroute/storage.sqlite`（腳本以 `mode=ro` + `PRAGMA query_only=1` 開啟，不會與 OmniRoute 自身寫入衝突）。

## 安裝步驟

1. **確認 Python 路徑**（Windows 原生 python.exe，不是 Git Bash 的 python3 別名）：
   ```powershell
   Get-Command python -ErrorAction SilentlyContinue
   ```

2. **複製腳本到穩定位置**（此 skill 已內建一份現成腳本 `scripts/statusline.py`；若專案目錄 `D:\CODES\omniroute-hud\` 已存在同名腳本，可直接沿用，兩者內容相同）：
   ```
   複製 C:\Users\HCH\.claude\skills\omniroute-hud\scripts/statusline.py
   到  D:\CODES\omniroute-hud\scripts/statusline.py   （或任何你偏好的固定路徑）
   ```

3. **寫入 `~/.claude/settings.json` 的 `statusLine` 區塊**（路徑需替換成實際 python.exe 與腳本位置，Windows 路徑用正斜線並跳脫雙引號）：
   ```json
   "statusLine": {
     "type": "command",
     "command": "\"C:/Users/HCH/AppData/Local/Programs/Python/Python312/python.exe\" \"D:/CODES/omniroute-hud/scripts/statusline.py\""
   }
   ```
   若 `settings.json` 已有其他 `statusLine`（例如 `statusline-setup` skill 裝的雙帳號版本），兩者互斥 — 只能存在一個 `statusLine` key，需自行決定要留哪一個。

4. **重啟 Claude Code** 讓 `settings.json` 變更生效。

## 驗證

1. 自我測試（純格式邏輯，不連線、不讀 DB）：
   ```
   "<python.exe>" "<script>" --selftest
   ```
   預期輸出：`selftest OK`

2. 模擬 stdin 測試（需要一個 Windows 路徑下的 JSON 檔，Git Bash 的 `/tmp` 路徑對原生 python.exe 不可見）：
   ```json
   {"cwd":"C:\\Users\\HCH\\project","model":{"display_name":"Sonnet 5"},"context_window":{"used_percentage":42}}
   ```
   ```
   "<python.exe>" "<script>" < test_stdin.json
   ```
   OmniRoute 存活時預期輸出類似（固定兩行；第二行列出每個已註冊、且有 quota 快照的帳號，不限 provider，目前使用中的帳號標 `*`，有 `reset_at` 資料的額度會附註距離重置還有多久）：
   ```
   project | Sonnet 5 | ctx:42% | OmniRoute:up claude/claude-sonnet-5
   hch590902:5h:23%(→2h10m) 7d:0%(→1d16h) hch.new*:5h:57%(→4h0m) 7d:59%(→4d3h) deepseek:main:credit:100%
   ```

3. 成功標準：OmniRoute 服務關閉時應顯示 `OmniRoute:down`（紅色）而非拋出例外或空白列。

## 已知限制

- 額度數字來自 OmniRoute 背景輪詢寫入 `quota_snapshots` 的快取值，非即時查詢 provider API，可能有分鐘級延遲。
- `OMNIROUTE_DB` 環境變數可覆寫預設 DB 路徑（預設 `~/.omniroute/storage.sqlite`），供多套 OmniRoute 安裝或測試情境使用。
- 與 `statusline-setup` skill（Anthropic 帳號雙帳號額度/git/MCP chip 狀態列）功能不同、彼此獨立，但 `settings.json` 的 `statusLine` 只能設定一組 — 兩者不能同時生效。

## 維護

此 skill 內建的 `scripts/statusline.py` 與研發原始碼（`D:\CODES\omniroute-hud\scripts/statusline.py`）內容一致。若之後修改主腳本，記得同步複製回 `C:\Users\HCH\.claude\skills\omniroute-hud\scripts/statusline.py`，讓此 skill 保持可獨立重裝。

---

## Conformance Addendum

## When to Use
Install a Claude Code status line that shows OmniRoute gateway liveness, the currently-active provider/model, remaining quota for EVERY registered provider connection across all providers — Claude, DeepSeek, etc. (session 5h / weekly 7d / credits_usd), broken connections, and phantom/ghost error counts. Reads OmniRoute's local SQLite DB (storage.sqlite) read-only plus a raw TCP liveness check on port 20128 — no HTTP auth needed. The fixed, ready-to-use script is bundled in this skill. Use when the user wants to see OmniRoute status or remaining model quota (including multi-account, multi-provider setups) directly in the Claude Code status line.

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
