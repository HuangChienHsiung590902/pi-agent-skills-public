---
name: obsidian-mcp-skill-router
description: 在回答使用者問題前，先透過 MCP/Obsidian 或 D:\OB\skills 索引搜尋是否有合適 skill 可用；找到候選後必須讀取對應 SKILL.md 再執行。用在幾乎所有非純聊天、非一次性知識問答的任務，尤其是本機工具、MCP、Agent、ECP/aipower、Obsidian、ComfyUI、Docker、瀏覽器自動化、Windows 設定與除錯。
description_zh: 回答前自動查 Obsidian skills library，判斷是否有可套用技能，避免憑記憶亂做。
---

# Obsidian MCP Skill Router

## When to Use

每次使用者提出「需要操作、設定、修復、開發、部署、查本機資料、控制工具、呼叫 MCP、整理文件、除錯」這類任務時，都先使用本 skill。

特別是使用者提到以下任一關鍵字時必用：

- MCP、Claude Code、Pi、OpenCode、OmniRoute、agent、skills
- Obsidian、vault、筆記、知識庫、搜尋
- ECP、aipower、Chainsea、CBM、LINE、VRS
- ComfyUI、影像、影片、語音、Whisper、GPU
- Docker、SSH、Cloudflare Tunnel、FRP、ESXi、遠端主機
- Playwright、Chrome、CDP、瀏覽器、自動化
- Windows、PowerShell、symlink、junction、設定檔搬移
- AnyTXT、OCR、PDF、DOCX、XLSX、文件搜尋
- skill-update（把本回合過程寫進現有 skill）

不用於：

- 使用者只是在閒聊、問一般常識，且不需要本機流程或工具經驗。
- 已經在同一回合讀過明確匹配的 skill，且任務範圍未改變。

## Procedure

1. **把使用者問題轉成搜尋關鍵字**
   - 抽出工具名、專案名、錯誤關鍵字、路徑、任務動詞。
   - 同時準備中英文同義詞，例如：
     - `瀏覽器` / `browser` / `chrome` / `cdp` / `playwright`
     - `知識庫` / `knowledge` / `obsidian` / `vault`
     - `修復` / `debug` / `troubleshoot` / `fix`

2. **優先直接查本機檔案系統索引**
   - `D:\OB\skills` 是 skill library 的實際落地來源；查 skill 不需要先連 Obsidian MCP。
   - 直接讀或搜尋：
     ```text
     D:\OB\skills\SKILLS_INDEX.md
     D:\OB\skills\_consolidation\SKILLS_CATALOG.json
     D:\OB\skills\_consolidation\SKILLS_TAG_INDEX.md
     D:\OB\skills\_consolidation\SKILLS_SEARCH_GUIDE.md
     D:\OB\skills\*\SKILL.md
     ```
   - 可用搜尋指令：
     ```powershell
     rg -n -i "<keyword>" D:/OB/skills/SKILLS_INDEX.md D:/OB/skills/_consolidation D:/OB/skills/*/SKILL.md
     ```
   - 或用 PowerShell 查 catalog：
     ```powershell
     $cat = Get-Content -Raw D:/OB/skills/_consolidation/SKILLS_CATALOG.json | ConvertFrom-Json
     $cat.skills | Where-Object { ($_.id + ' ' + $_.description + ' ' + $_.annotation + ' ' + ($_.tags -join ' ')) -match '<keyword>' } | Select-Object id,category,description
     ```
   - 若只是在找、讀、套用 skill，這個本機流程是預設路徑，避免 Obsidian MCP timeout 阻塞任務。

3. **只有需要 Obsidian app/vault 狀態時才改用 Obsidian MCP**
   - 適用情境：需要 Obsidian 的 active file、workspace、vault link/tag 語意、插件搜尋能力，或要透過 Obsidian 寫入/操作 note。
   - 先看 MCP server 狀態：
     ```js
     mcp({ server: "obsidian" })
     ```
   - 若尚未連線，嘗試：
     ```js
     mcp({ connect: "obsidian" })
     ```
   - 不要假設 Obsidian MCP tool 名稱固定；先列工具或 describe，再呼叫可搜尋/讀檔/寫入的 tool。
   - Obsidian MCP 不可用或 timeout 時，不要放棄找 skill；立刻回到第 2 步的 `D:\OB\skills` 檔案系統索引。

4. **挑選候選 skill**
   - 優先順序：
     1. `description` 或 `When to Use` 明確命中任務。
     2. skill 名稱與工具/專案完全匹配。
     3. tags/category 與任務相符。
     4. 搜尋結果多次命中相同 skill。
   - 通常挑 1–3 個候選即可；不要把整個 catalog 塞進上下文。
   - **不要用 OpenJEV（`10.145.119.19:8090`）當主路由。** 它是 NLI 判定器，不是 skill 選擇器；比本機 catalog/`rg` 更慢、無參考 rerank 近乎隨機。決策見 `skills-openjev-routing`。只有使用者明確要求、且關鍵字已縮到 3–5 個高度重疊候選時，才可當可選第二段打分，分數不能取代讀完整 `SKILL.md`。

5. **必須讀取候選 `SKILL.md` 後才套用**
   - 找到候選後，讀：
     ```text
     D:\OB\skills\<skill-id>\SKILL.md
     ```
   - 不要只根據 skill 名稱或 catalog 摘要執行。
   - 若 skill 內有相對路徑，路徑基準是該 `SKILL.md` 所在資料夾。

6. **回覆/執行時簡短標註使用到的 skill**
   - 若有載入 skill，回覆中可簡短寫：
     ```text
     我先查了 skills，這題匹配 <skill-id>，以下依照該流程處理。
     ```
   - 若沒有找到合適 skill，簡短說：
     ```text
     我查過 Obsidian skills，目前沒有明確匹配，改用一般流程處理。
     ```

7. **如果發現常見任務沒有 skill，主動建議補一個**
   - 不要每次都立即建立；除非使用者要求，或任務流程明顯可重用。
   - 新 skill 位置固定：
     ```text
     D:\OB\skills\<lowercase-kebab-slug>\SKILL.md
     ```
   - 建完後更新：
     ```text
     D:\OB\skills\SKILLS_INDEX.md
     D:\OB\skills\_consolidation\*
     ```

8. **使用者訊息是 `skill-update`（不要斜線）**
   - 把本回合過程與結果更新到 `D:\OB\skills` 裡最相關的**現有** skill，不要新建重複的。
   - 先出執行摘要，等 y 再改檔。
   - `/skill-` 是 Pi 內建 skill 選單，裡面不會出現 skill-update。

## Pitfalls

- 不要把「MCP tool」和「skill」混為一談：MCP tool 是可呼叫工具；skill 是 `D:\OB\skills\...\SKILL.md` 的流程說明。
- Obsidian MCP 可能 timeout；這不是 skills 不存在。必須 fallback 到 `D:\OB\skills` 檔案索引。
- 不要只查 `C:\Users\HCH`。此機器許多 agent/Claude/Pi 設定已搬到 `D:\.system`，`C:\Users\HCH` 常是 junction/symlink 相容層。
- 不要只憑 catalog 的一句 description 就修改系統；要讀完整 `SKILL.md`。
- 搜尋結果太多時，先縮小到工具名 + 任務動詞，例如 `mcp timeout`、`ecp menu`、`comfyui lora`。相近 skill 分不清時，優先改各 skill 的 `description` / `When to Use` 寫互斥，不要改接 OpenJEV。
- 不要把 OpenJEV `/rerank` 或 `/predict` 當成選 `SKILL.md` 的預設步驟（見 `skills-openjev-routing`）。
- 若目前 session 的 system prompt 沒列出新 skill，不代表它不存在；仍可直接讀 `D:\OB\skills\obsidian-mcp-skill-router\SKILL.md`。
- 使用者打 `skill-update` 是普通訊息，不是 `/skill-update`。斜線選單只列出已載入 skill（目前通常只有 router）。

## Verification

1. `D:\OB\skills\obsidian-mcp-skill-router\SKILL.md` 存在。
2. `D:\OB\skills\SKILLS_INDEX.md` 或 `_consolidation/SKILLS_CATALOG.json` 能搜尋到 `obsidian-mcp-skill-router`。
3. `D:\.system\.claude\skills\obsidian-mcp-skill-router\SKILL.md` 存在，證明 Claude/Pi junction 可看到它。
4. 實際遇到新問題時，agent 會先查 Obsidian skills，再讀取匹配的 `SKILL.md`，最後才回答或操作。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
