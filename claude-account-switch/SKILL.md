---
name: claude-account-switch
description: >-
  Claude Code 多帳號切換的兩種機制總覽與安裝——(1) `ccs` 憑證熱切換（同一個 session 內換帳號，推薦，`scripts/install.sh`
  在本目錄）(2) `ccc`/`ccc-watch`/`ccc-resume2` relay 套件（重啟 TUI 換帳號並搬移對話）。合併自原本的
  `ccc-switch-setup` 與 `ccs-install` 兩個 skill——過去分成兩篇的原因是各自獨立演進，但兩者服務同一個目標
  （多帳號、rate limit 自動應對），現在統一維護。
triggers:
  - ccc
  - ccc-switch
  - ccs
  - 帳號切換
  - 自動切換帳號
  - rate limit 切換
  - 多帳號切換
  - credential hot-swap
---

# Claude Code 多帳號切換（ccs 熱切換 + ccc relay 套件）

兩種模式，**選一種用，不要混著用**：

| 模式 | 心智模型 | 何時用 |
|---|---|---|
| **`ccs`（推薦 ⭐）** | 同一個 Claude Code session 持續開著，只換底層憑證檔案；下一則訊息就切到新帳號，不重啟、不搬對話 | 一般日常用，想要無縫切換 |
| **`ccc` 套件** | 重啟 TUI，把對話 `.jsonl` 複製到另一帳號的 config dir 再 resume | 需要兩個帳號各自獨立的 `~/.claude` / `~/.claude-2` config（skills/settings 不共用）時 |

---

## 模式一：`ccs` 憑證熱切換（推薦）

### 安裝

```bash
# 基本安裝（需手動改 email）
bash ~/.claude/skills/claude-account-switch/scripts/install.sh \
  --email1 帳號1@gmail.com \
  --email2 帳號2@gmail.com

# 指定腳本安裝目錄（預設 ~/bin）
bash ~/.claude/skills/claude-account-switch/scripts/install.sh \
  --email1 帳號1@gmail.com \
  --email2 帳號2@gmail.com \
  --bin /d/BIN
```

安裝腳本（`scripts/install.sh`，本目錄，完全自包含、無外部依賴）會：
1. 寫入 `ccs` 到指定 bin 目錄
2. 寫入 `~/.claude/hooks/ccs-autoswap.sh`
3. 在 `settings.json` 加入 `UserPromptSubmit` hook
4. 建立 `~/.claude-accounts/` 並設 active=1

### 安裝後

```bash
# 1. 確認 PATH
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 2. Seed 兩個帳號的憑證
#    帳號1 用 claude 登入後：
cp ~/.claude/.credentials.json ~/.claude-accounts/1.credentials.json
#    帳號2（CLAUDE_CONFIG_DIR=~/.claude-2 claude，/login）後：
cp ~/.claude-2/.credentials.json ~/.claude-accounts/2.credentials.json

# 3. 驗證
ccs status
```

### 元件說明

| 元件 | 路徑 | 功能 |
|------|------|------|
| `ccs` | `~/bin/ccs`（本機實際在 `/d/BIN/ccs`，`~/bin` 已不存在） | 手動切帳號：`ccs` toggle、`ccs 1/2`、`ccs status` |
| `ccs-autoswap.sh` | `~/.claude/hooks/` | 自動切：Path A 偵測 limit 錯誤、Path B 用量 ≥ 80% |
| 憑證庫 | `~/.claude-accounts/{1,2}.credentials.json` | 各帳號憑證存放 |
| active marker | `~/.claude-accounts/active` | 記錄目前生效帳號（1/2） |
| `~/.claude-accounts/.last-swap` | Cooldown 時間戳 |
| `~/.claude-accounts/autoswap.log` | 每次自動切換決策的 append-only log（HARD-BLOCK/SOFT-BLOCK/SWAP/SKIP） |
| `statusline-command.sh` | `.claude` 目錄下，`①/②` 標記跟著 `active` marker 走 |

### 手動使用（session 內，不用開新終端機）

```bash
!ccs        # toggle 1↔2 — 下一則訊息生效
!ccs 2      # 切到帳號 2
!ccs status # 顯示目前狀態
```

### 自動切換（零按鍵）

`UserPromptSubmit` hook 在你的訊息被處理**之前**執行，兩個觸發條件任一成立就切換：

- **Path A — 硬訊號（可靠）**：讀取 hook stdin 的 `transcript_path`，檢查**最後一則 assistant 回合**是否為
  `isApiErrorMessage:true` 且含 limit 文字（`hit your … limit / usage limit / session limit / rate limit /
  plan limit / over_capacity`）。若是，**無視百分比強制切換**——這是 ground truth，能抓到「訊息middle就撞到
  limit」的情況（此時 5h 視窗可能已經重置成低%，百分比判斷法會漏掉）。
- **Path B — 主動百分比（提早留margin）**：讀取現役帳號最新 5h 用量（來自 statusline 的
  `~/.claude-quota-cache/<active>.json`），達到門檻就提早切換，不用等真的撞牆。

環境變數：

| 變數 | 預設 | 說明 |
|------|------|------|
| `CCS_AUTOSWAP` | `1` | `0` 停用自動切換 |
| `CCS_THRESHOLD` | `80` | Path B 觸發門檻（用量 %） |
| `CCS_COOLDOWN` | `180` | 兩次自動切換最短間隔（秒） |

> Path B 需要 statusline 寫入 `~/.claude-quota-cache/{1,2}.json`；無 statusline 時只有 Path A 生效。
> `.bashrc` 的 export 不會傳到非互動的 hook——要改預設值請改 script 內建值，或透過
> `settings.json → env`（Claude Code 會把這個傳給 hooks）。

### 排錯：狀態列額度跟網頁「差很多」/ autoswap 一直失敗

| 症狀 | 根因 | 修法 |
|------|------|------|
| `autoswap.log` 一直 `SWAP X → Y 失敗（ccs 回非零）` | `ccs` 不在 PATH（安裝目錄可能已搬過，如 `~/bin` 換成 `/d/BIN`） | 重新跑本 skill 的 `scripts/install.sh --bin <目前實際目錄>`，或直接 `chmod +x` 確認執行權限 |
| 狀態列 ①/② 額度與 claude.ai 網頁 Usage 對不上 | swap 換了憑證但 `active` marker 沒同步 → 把現役帳號的額度貼到另一個 badge 上 | 比對「現役 session 回報的 5h reset 時間」與網頁 reset 時間：一個 5h 視窗不可能有兩個重置時間，對不上就代表 marker 指錯帳號。修 `~/.claude-accounts/active` 指向真正生效的帳號 |
| 某帳號額度顯示是舊的測試值（如 35%/18%） | `~/.claude-quota-cache/{1,2}.json` 殘留安裝時的範例資料 | 該帳號實際跑一次狀態列即覆蓋；或手動寫入正確值（存的是 **used%**，腳本顯示 `100-used` 為剩餘） |

> **絕不**手動編輯 `.credentials.json` 來「修額度」——額度是雲端帳號狀態，本機只能改顯示快取與 marker。

若 `ccs` 整支從系統消失（不只是排錯，是檔案真的不在了），直接重跑本 skill 開頭的 `scripts/install.sh` 重建，
不需要手動重打原始碼。

### 已知限制

- 依賴 Claude Code 在 session 中重新讀取 `.credentials.json`（Windows/Linux 有文件記載此行為；macOS 快取約 30 秒）。
  第一次用完 `ccs` 後驗證一次：下一則訊息的帳號 / statusline 的 5h 數字應該反映另一個帳號。
- **Path B**（主動%）依賴 statusline 最近有渲染過（它會寫入 quota cache 給 hook 讀）——實務上因為
  statusline 持續渲染所以成立。**Path A**（硬訊號）不依賴 cache——直接讀 transcript，所以即使 cache 過期
  或顯示誤導性的低% 時仍能觸發，這是 Path A 撐住 blind spot 的關鍵。
- 可以跟 `ccc-watch` relay 並存：純 `claude`（一個 config dir）→ 走憑證熱切換；用 `ccc` 啟動（兩個 config
  dir）→ 走 relay。選一種模式，不要同時用。

---

## 模式二：`ccc` relay 套件（重啟 TUI + 對話搬遷）

Windows + Git Bash 專用的多帳號 launcher 套件。

### 各腳本作用

| Script | Purpose |
|--------|---------|
| `ccc` | Interactive launcher menu (fzf) — pick account or relay mode |
| `ccc-watch` | Wraps `claude`; detects rate limit on exit and prompts to switch accounts |
| `ccc-resume2` | Copies a conversation's `.jsonl` to the other Claude account and resumes it |
| `ccc-mirror-config` | Symlinks account 1's settings/skills/hooks into account 2 (auto-sync) |
| `ccc-codex-resume` | Same as `ccc-resume2` but for Codex accounts |
| `ccc-cross-relay` | Claude ↔ Codex cross-tool relay with handoff capsule |

Source directory: `D:\Docs\Claude x 2\ccc-switch\`

### Prerequisites

```bash
fzf --version   # required — interactive menu
claude --version
```

### Step 1 — Install fzf (Windows)

```powershell
winget install --id junegunn.fzf --accept-source-agreements --accept-package-agreements --silent
```

```bash
mkdir -p ~/bin
cp "/c/Users/$USERNAME/AppData/Local/Microsoft/WinGet/Links/fzf.exe" ~/bin/fzf.exe
hash -r
fzf --version   # should print 0.x.x
```

### Step 2 — Run the install script

```bash
bash "/d/Docs/Claude x 2/ccc-switch/scripts/install.sh"
```

The script auto-detects what's installed:
- `claude` found → installs `ccc`, `ccc-resume2`, `ccc-mirror-config`, `ccc-watch`
- `codex` found → also installs `ccc-codex-resume`
- Both found → also installs `ccc-cross-relay`

All scripts land in `~/bin/`.

### Step 3 — Set environment variable

Add to `~/.bashrc` (create if it doesn't exist):

```bash
export PATH="$HOME/bin:$PATH"
export CLAUDE_CONFIG_DIR_2="$HOME/.claude-2"
```

Verify:
```bash
source ~/.bashrc
echo $CLAUDE_CONFIG_DIR_2   # should print /c/Users/HCH/.claude-2
which ccc                    # should print ~/bin/ccc
```

### Step 4 — Log in to account 2

Must be done in a **real terminal** (not via `!` inside Claude Code — that runs non-interactive):

```powershell
$env:CLAUDE_CONFIG_DIR = "$HOME\.claude-2"; claude
```

Then run `/login` inside that Claude Code session using `hch.new@gmail.com`.

Verify login succeeded:
```bash
jq '.oauthAccount.emailAddress' ~/.claude-2/.claude.json
```

### Step 5 — Mirror config from account 1 to account 2

```bash
ccc-mirror-config
```

Symlinks these items from `~/.claude` into `~/.claude-2`:

```
settings.json  settings.local.json  statusline-command.sh
rules  skills  commands  agents  hooks  scripts
```

Items that stay **independent** per account: login credentials, conversation history (`projects/`), sessions, plugins, cache.

> **Re-run `ccc-mirror-config` any time you add new config items** to account 1. Idempotent — existing symlinks skipped, old files backed up with a timestamp suffix.

### Usage

#### Normal launch
```bash
ccc
```
fzf menu:
```
Claude Code (1)  · 主帳號
Claude Code (2)  · 第二帳號
Claude 接力 · 跨帳號 resume 同一場對話 (1↔2)
```

#### Rate limit auto-switch flow (fully automatic — no prompt)

1. Start via `ccc` → runs `env CCC_ACCOUNT=N … ccc-watch $FLAGS`
2. Claude hits rate limit and **exits non-zero**
3. `ccc-watch` scans the last 30 lines of the session `.jsonl` for: `rate limit / too many requests / 429 / usage limit / plan limit / over_capacity`
4. If matched → **auto-relays** the conversation to the other account (non-interactive `ccc-resume2 --to <other> --session <id> --yes --no-launch`) and **re-launches it under `ccc-watch` again** — so ping-pong when both accounts limit out.
5. **Loop guard:** each no-progress switch increments `CCC_SWITCH_COUNT`; running ≥ 60s resets it. At `CCC_MAX_SWITCH`（default 4）it stops with「兩個帳號似乎都已用盡」.

Knobs (env):
- `CCC_AUTOSWITCH=0` → revert to `(Y/n)` confirmation.
- `CCC_MAX_SWITCH=N` → default 4.

> **Hard requirement:** auto-switch only triggers when Claude Code **exits** on the limit. If it just shows the usage-limit message and **stays open**, `ccc-watch` never regains control — quit (Ctrl-C/exit) to trigger relay.

#### Manual cross-account relay — interactive
```bash
ccc-resume2 [path]
```

#### Manual cross-account relay — non-interactive
```bash
ccc-resume2 --to 1|2 [--session ID] [--yes] [--launch|--no-launch] [--path DIR]
```
- launch behaviour: **auto**（TTY 存在才 launch）；無 TTY（例如 `!ccc-resume2 …`）只複製對話並印出 resume 指令。

#### Cross-tool relay (Claude ↔ Codex)
```bash
ccc-cross-relay [path]
```
產出 `.ccc-handoff.md` capsule（原始任務、最後請求、最後進度、改動檔案、git diff），啟動目標工具並預載 handoff prompt。

### How ccc-watch detects rate limits

```bash
grep -qiE 'rate.?limit|too many requests|429|usage.?limit|plan limit|over_capacity'
# 掃 ~/.claude/projects/<proj>/<latest>.jsonl 最後 30 行
```
啟發式判斷——若 Claude 因其他原因退出但含相符關鍵字，可能誤判，實務上少見。

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `CLAUDE_CONFIG_DIR_2` | `~/.claude-2` | Account 2 config directory |
| `CCC_CLAUDE_FLAGS` | `--dangerously-skip-permissions` | Extra flags passed to `claude` |
| `CODEX_ACCOUNT_HOMES_DIR` | `~/.codex-homes` | Root dir for multiple Codex accounts |

### Known pitfalls

| Pitfall | Fix |
|---------|-----|
| `fzf` not on bash PATH after winget | Copy to `~/bin` |
| `CLAUDE_CONFIG_DIR=~/.claude-2 claude` fails in PowerShell | Use `$env:CLAUDE_CONFIG_DIR = "$HOME\.claude-2"; claude` |
| `!` inside Claude Code is non-interactive — can't launch account 2 login | Open a real terminal window |
| `ccc-mirror-config` overwrites account 2's custom settings | Backup is auto-created as `<file>.bak-<timestamp>` |
| `ccc-watch` false-positive rate limit detection | Check the `.jsonl` manually |

---

## User-specific config (this machine)

- Account 1: `hch590902@gmail.com` → `~/.claude`
- Account 2: `hch.new@gmail.com` → `~/.claude-2`
- `ccc` 系列 scripts 安裝到：`~/bin/`
- `ccs` 實際安裝在 `/d/BIN/ccs`（`~/bin` 已不存在，工具都在 `/d/BIN`，在 PATH 上）
- `.bashrc` 設定 `CLAUDE_CONFIG_DIR_2`、`PATH`、`CCC_CLAUDE_FLAGS`、`alias claude`

### ~/.bashrc (current state)
```bash
export PATH="$HOME/bin:$PATH"
export CLAUDE_CONFIG_DIR_2="$HOME/.claude-2"
export CCC_CLAUDE_FLAGS="--dangerously-skip-permissions"
alias claude='claude --dangerously-skip-permissions'
```

---

## Conformance Addendum

## When to Use
Claude Code 多帳號切換的兩種機制總覽與安裝——(1) `ccs` 憑證熱切換（同一個 session 內換帳號，推薦，`scripts/install.sh` 在本目錄）(2) `ccc`/`ccc-watch`/`ccc-resume2` relay 套件（重啟 TUI 換帳號並搬移對話）。合併自原本的 `ccc-switch-setup` 與 `ccs-install` 兩個 skill——過去分成兩篇的原因是各自獨立演進，但兩者服務同一個目標 （多帳號、rate limit 自動應對），現在統一維護。

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
