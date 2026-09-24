---
name: dotfiles-to-d-system
description: 'Consolidate every agent/tool configuration directory on Windows (.pi, .claude, .mcp, .omniroute, .vscode, npm global prefix, npm cache, codegraph, …) into a single D:\.system folder, leaving NTFS junctions behind so all existing tools keep working from their original paths. Designed so the machine can run Deep Freeze (or any C: drive restore/reset software) without losing agent settings, OAuth credentials, quota history, skills, or MCP configuration — and so the user only ever has to back up ONE folder. Includes the safe move procedure (count verification, DB backup, service stop/restart), the npm prefix/cache relocation, User PATH rewrite, and the gotchas that silently break things (UTF-8 BOM in JSON, Remove-Item on junctions, hardcoded paths inside hud.ts / settings.json / mcp.json). Use when the user wants to move dotfiles off C:, prepare for Deep Freeze, centralize configs, or asks why settings disappear after reboot.'
triggers:
  - deep freeze
  - deepfreeze
  - 冰點還原
  - 設定檔搬到 d 槽
  - dotfiles 搬移
  - 'd:\.system'
  - 集中設定
  - junction 設定
  - npm prefix 搬家
  - 重開機設定消失
  - 只備份一個資料夾
---

# Dotfiles → `D:\.system`（Deep Freeze 友善的設定集中化）

把 Windows 上所有 agent / 工具的設定目錄集中到單一資料夾 `D:\.system`，原路徑改留 **NTFS junction**。這樣：

- C 槽可以安心用 Deep Freeze 凍結，重開機還原也不會弄丟設定
- 所有工具**完全不用改設定**，仍走原本的 `~/.pi`、`~/.claude` 等路徑
- 使用者**只需要備份 `D:\.system` 一個資料夾**

> 本 skill 記錄的是**已在此機器實際執行並驗證過**的完整流程與踩雷點。

## 最終架構

```text
D:\.system\
├── .claude          ← Claude skills / CLAUDE.md
├── .codegraph       ← codegraph 遙測
├── .conda
├── .config
├── .dsh             ← ★ DeepSeek dsh agent 設定、sessions、workspace
├── .local
├── .mcp             ← MCP servers（comfyui-mcp、playwright-mcp、npm）
├── .omc             ← oh-my-claudecode
├── .omniroute       ← ★ OmniRoute DB、OAuth、額度快照、logs
├── .pi              ← ★ pi agent 設定、extensions、sessions
├── .vscode
├── .vscode-shared
├── npm              ← npm global prefix
├── npm-cache        ← npm cache
├── codegraph        ← codegraph 執行檔
└── _backup          ← 各步驟自動備份
```

C 槽對應（**全部都是 junction，無真實資料**）：

```text
C:\Users\HCH\.claude                     -> D:\.system\.claude
C:\Users\HCH\.codegraph                  -> D:\.system\.codegraph
C:\Users\HCH\.comfyui-mcp                -> D:\.system\.mcp\comfyui-mcp
C:\Users\HCH\.conda                      -> D:\.system\.conda
C:\Users\HCH\.config                     -> D:\.system\.config
C:\Users\HCH\.dsh                        -> D:\.system\.dsh
C:\Users\HCH\.local                      -> D:\.system\.local
C:\Users\HCH\.omc                        -> D:\.system\.omc
C:\Users\HCH\.omniroute                  -> D:\.system\.omniroute
C:\Users\HCH\.pi                         -> D:\.system\.pi
C:\Users\HCH\.playwright-mcp             -> D:\.system\.mcp\playwright-mcp
C:\Users\HCH\.vscode                     -> D:\.system\.vscode
C:\Users\HCH\.vscode-shared              -> D:\.system\.vscode-shared
C:\Users\HCH\AppData\Local\codegraph     -> D:\.system\codegraph
```

## 為什麼要留 junction（不要直接改路徑就好）

很多程式硬寫 `~/.pi`、`~/.claude`、`%APPDATA%\npm`，有些甚至寫在套件內部無法設定。junction 讓**實體資料集中**的同時**保持所有舊路徑可用**，是最穩的折衷。

## ⚠️ 絕對不要做的事

**不要把整個 `C:\Users\<user>` junction 到 D 槽。**

`NTUSER.DAT`（登錄檔 hive）在登入時就被系統鎖定並載入，而 profile 載入發生在檔案系統 junction 生效之前，D 槽當下可能還沒就緒。弄壞的後果是**無法登入 Windows**。

Microsoft 官方只支援兩種做法：安裝時用 `unattend.xml` 指定 `ProfilesDirectory`，或搬移個別「已知資料夾」。**不支援**把現有 profile 整包搬走。

正確做法就是本 skill 的**逐個目錄 junction**。

## 標準搬移程序

每個目錄都照這個順序，缺一不可：

1. **確認來源不是 junction**（避免重複搬移）
2. **確認目的地不存在或為空**（避免覆蓋）
3. **記錄搬移前的檔案數 / 資料夾數**
4. `Move-Item` 搬移
5. **比對搬移後數量**，不符就 `throw`
6. `New-Item -ItemType Junction` 建立連結
7. **讀寫穿透測試**：從 C 路徑寫檔，確認落在 D

參考實作：

```powershell
$src = "C:\Users\HCH\$name"
$dst = "D:\.system\$name"

$i = Get-Item -LiteralPath $src -Force
if ($i.Attributes -band [IO.FileAttributes]::ReparsePoint) {
    Write-Host "already a link"; return
}

$beforeF = @(Get-ChildItem -LiteralPath $src -Recurse -Force -File      -ErrorAction SilentlyContinue).Count
$beforeD = @(Get-ChildItem -LiteralPath $src -Recurse -Force -Directory -ErrorAction SilentlyContinue).Count

Move-Item -LiteralPath $src -Destination $dst

$afterF = @(Get-ChildItem -LiteralPath $dst -Recurse -Force -File      -ErrorAction SilentlyContinue).Count
$afterD = @(Get-ChildItem -LiteralPath $dst -Recurse -Force -Directory -ErrorAction SilentlyContinue).Count
if ($beforeF -ne $afterF -or $beforeD -ne $afterD) { throw "Count mismatch" }

New-Item -ItemType Junction -Path $src -Target $dst | Out-Null
```

## 踩過的雷（重要）

### 1. `Remove-Item` 刪不掉 junction

PowerShell 5.1 對 junction 呼叫 `Remove-Item` 會丟 `NullReferenceException`。

**改用 `cmd /c rmdir`**（只刪連結，不會刪到目標內容）：

```powershell
cmd.exe /c rmdir "$path" | Out-Null
```

### 2. `Set-Content -Encoding UTF8` 會寫入 BOM，讓 JSON 解析失敗

改寫 `settings.json` / `mcp.json` 後，pi 會噴：

```text
Warning: (global settings) Unexpected token ... is not valid JSON
```

**改用 Node 重寫**，或明確寫出無 BOM：

```javascript
let buf = fs.readFileSync(f);
if (buf[0] === 0xEF && buf[1] === 0xBB && buf[2] === 0xBF) buf = buf.subarray(3);
const obj = JSON.parse(buf.toString('utf8'));
fs.writeFileSync(f, JSON.stringify(obj, null, 2) + '\n', { encoding: 'utf8' });
```

### 3. 設定檔裡有硬寫路徑，搬完必須一起改

搬完務必用 ripgrep 全掃一次：

```powershell
& "D:\.system\.pi\agent\bin\rg.exe" -n --hidden -g "!node_modules/**" -g "!sessions/**" `
  "D:[/\\]\.(pi|claude|mcp)" "D:\.system"
```

此機實際命中的：

| 檔案 | 欄位 |
|---|---|
| `.pi\agent\settings.json` | `skills` |
| `.pi\agent\mcp.json` | 兩個 MCP server 的 `args` 路徑 |
| `.pi\agent\extensions\hud.ts` | `better-sqlite3` 載入路徑 |

### 4. `hud.ts` 硬寫 `%APPDATA%\npm` 載入 better-sqlite3

npm prefix 搬走後 HUD 額度顯示會直接壞掉。**改成多路徑備援**：

```typescript
const candidates = [
  process.env.PI_SQLITE_HOST_PKG,
  "D:/.system/.mcp/npm/package.json",
  path.join(home, ".mcp", "npm", "package.json"),
  "D:/.system/npm/node_modules/omniroute/package.json",
  path.join(process.env.APPDATA || "", "npm", "node_modules", "omniroute", "package.json"),
].filter(Boolean) as string[];

for (const pkgPath of candidates) {
  try {
    return createRequire(pkgPath)("better-sqlite3");
  } catch { /* try next */ }
}
```

> 註：此機唯一可用的 `better_sqlite3.node` 在 `D:\.system\.mcp\npm\node_modules\`，npm global 那份**沒有編譯出原生模組**。

### 5. `.omniroute` 必須先停服務再搬

`storage.sqlite` 被 OmniRoute 開著，直接搬可能損毀 DB。

程序：**停 port 20128 → 等 port 釋放 → 用獨佔模式測試檔案未鎖 → 備份 DB → 搬移 → 建 junction → 重啟服務**。

腳本已備妥：`D:\.system\move-omniroute.ps1`

執行時 pi / Claude Code 走 OmniRoute 的連線會中斷，所以**要在一般 PowerShell 視窗執行**，不要在 agent 內執行。

## npm 遷移（獨立流程）

npm 不用 junction，**直接改設定 + 重裝**比搬 14 萬個檔案乾淨。

```powershell
npm config set prefix "D:\.system\npm"
npm config set cache  "D:\.system\npm-cache"

npm install -g "@earendil-works/pi-coding-agent" "@colbymchenry/codegraph" "omniroute" --no-fund --no-audit
```

接著改 **User PATH**（登錄檔層級，非 session 變數）：

```powershell
$old = 'C:\Users\HCH\AppData\Roaming\npm'
$new = 'D:\.system\npm'
$p = [Environment]::GetEnvironmentVariable('PATH','User')
$p | Set-Content "D:\.system\_backup\userpath-$(Get-Date -f yyyyMMdd-HHmmss).txt"
$parts = $p -split ';' | Where-Object { $_ } | ForEach-Object {
    if ($_.TrimEnd('\') -ieq $old.TrimEnd('\')) { $new } else { $_ }
}
[Environment]::SetEnvironmentVariable('PATH', ($parts -join ';'), 'User')
```

### npm 相關注意事項

- PATH 改完，**現有終端機視窗仍是舊 PATH**，要開新視窗
- npm cache 是純快取，**直接刪即可**（此機釋放 2.8 GB），npm 會自己重建
- 舊 prefix 內的 `.node` 原生模組會被執行中的 pi / OmniRoute 鎖住，刪不掉是正常的 → **重開機後**再刪，腳本：`D:\.system\cleanup-old-npm-leftover.ps1`
- `npm install -g` 會跳過 install scripts（allow-scripts 機制），需確認原生模組是否真的可用，不要只看安裝成功

### bash 工具呼叫 npm 的坑

在 agent 的 bash 工具裡直接跑 `npm install -g @scope/pkg`，`@` 可能被吃掉導致參數錯亂；`cmd /c` 有時會掉進互動模式。

**改成直接呼叫 npm-cli.js**：

```powershell
& node 'C:\Program Files\nodejs\node_modules\npm\bin\npm-cli.js' install -g @pkgs --no-fund --no-audit
```

## 驗證清單

搬完務必逐項確認：

```powershell
# 1. C 槽已無真實 dotfile 目錄
Get-ChildItem 'C:\Users\HCH' -Force -Directory |
  Where-Object { $_.Name -like '.*' } |
  Select-Object Name,
    @{n='Type';e={ if ($_.Attributes -band [IO.FileAttributes]::ReparsePoint) {'Junction'} else {'REAL DIR ⚠'} }},
    @{n='Target';e={ $_.Target -join ';' }}

# 2. 指令解析到新位置
foreach ($n in 'pi','omniroute','codegraph') { (Get-Command $n).Source }

# 3. pi 可正常啟動（不應有 JSON 警告）
pi --offline -p "只回覆：OK" --no-tools

# 4. HUD 的 sqlite 相依可載入
node --experimental-strip-types --check "D:\.system\.pi\agent\extensions\hud.ts"

# 5. OmniRoute 服務正常
Invoke-WebRequest 'http://localhost:20128/home' -UseBasicParsing | Select-Object StatusCode
```

## Deep Freeze 使用須知

**凍結當下，所有 junction 與 PATH 必須已經設定正確。** 之後每次還原都會回到凍結那一刻的狀態。

凍結後**不要再做**這些事（做了也會被還原）：

- 改 User PATH
- `npm install -g` 裝新全域套件
- 在 C 槽新增 junction 或設定檔
- 改 `C:\Users\HCH\.npmrc`（**它是檔案不是目錄，junction 蓋不到**）

需要變更時：**先解凍 → 改 → 重新凍結**。

## 備份策略

只需要備份：

```text
D:\.system
```

**不要**備份 C 槽那些 junction，備份工具可能會跟著連結重複掃描，甚至誤判為循環。

## 相關檔案

| 路徑 | 用途 |
|---|---|
| `D:\.system\move-omniroute.ps1` | 停服務 → 備份 DB → 搬移 → 建 junction → 重啟 |
| `D:\.system\cleanup-old-npm-leftover.ps1` | 重開機後清理舊 npm prefix 殘留 |
| `D:\.system\_backup\` | DB、`.npmrc`、User PATH 的自動備份 |

## 相關 skills

- `omniroute-native` — OmniRoute 本機安裝與維運
- `omniroute-hud` — Claude Code statusLine 顯示額度
- `pi-coding-agent-setup` — pi agent 設定（HUD extension 在 `.pi\agent\extensions\hud.ts`）

---

## Conformance Addendum

## When to Use
Consolidate every agent/tool configuration directory on Windows (.pi, .claude, .mcp, .omniroute, .vscode, npm global prefix, npm cache, codegraph, …) into a single D:\.system folder, leaving NTFS junctions behind so all existing tools keep working from their original paths. Designed so the machine can run Deep Freeze (or any C: drive restore/reset software) without losing agent settings, OAuth credentials, quota history, skills, or MCP configuration — and so the user only ever has to back up ONE folder. Includes the safe move procedure (count verification, DB backup, service stop/restart), the npm prefix/cache relocation, User PATH rewrite, and the gotchas that silently break things (UTF-8 BOM in JSON, Remove-Item on junctions, hardcoded paths inside hud.ts / settings.json / mcp.json). Use when the user wants to move dotfiles off C:, prepare for Deep Freeze, centralize configs, or asks why settings disappear after reboot.

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
