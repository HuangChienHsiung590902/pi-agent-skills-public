---
name: claude-obsidian-skill-router
description: Claude Code 專用的本機 Skill 路由器。當使用者要求操作、設定、修復、開發、部署、查詢本機資料、控制 MCP/Agent、Obsidian、Windows、Docker、瀏覽器、ComfyUI、文件或其他需要專業流程的任務時，優先使用本 Skill：先搜尋 D:\OB\skills 的索引，再讀取最匹配的完整 SKILL.md，最後依該 Skill 執行。這不是 Pi 的啟動方式；不要把 `pi --no-skills --skill ...` 當成 Claude Code 指令。
compatibility: Claude Code user skills；本機正式 Skill library 位於 D:\OB\skills
---

# Claude Code × Obsidian Skill Router

## Purpose

本 Skill 是 **Claude Code 專用**的路由流程，將 `D:\OB\skills` 作為本機專業知識與操作流程的正式來源。它負責選擇並載入真正匹配的專業 Skill，不取代 Claude Code 的內建工具、MCP server 或一般推理能力。

## Claude Code 與 Pi 的差異

- **Pi 的用法（只適用 Pi）：**
  ```text
  pi --no-skills --skill D:/OB/skills/obsidian-mcp-skill-router/SKILL.md
  ```
- **Claude Code 不支援** `--no-skills --skill <path>` 這種 Pi 參數組合，也不要嘗試把它加入 `claude` 命令。
- **Claude Code 的用法：**讓本 Skill 放在 Claude Code 會自動發現的 user-skill 目錄中，依 frontmatter `description` 觸發；本機通常是：
  ```text
  C:\Users\Administrator\.claude\skills\claude-obsidian-skill-router\SKILL.md
  ```
  這個路徑目前透過 junction 指向正式來源 `D:\OB\skills`。
- 一般情況直接啟動：
  ```text
  claude
  ```
  Claude Code 會自動發現 user skills，符合任務描述時載入本 Skill。
- 若只想在單次 Claude Code session 額外注入路由提示，可使用 `--append-system-prompt-file <file>`；這是 system prompt 輔助，不是 `--skill` 啟動參數，也不會取代 Skill 自動發現。
- 不要因為 router 間接使用的 Skill 沒有直接 usage counter，就停用或刪除那些 Skill。

## When to Use

只要任務不是單純閒聊或一次性常識問答，且涉及下列任一情況，就先使用本 Skill：

- 需要操作、設定、修復、開發、部署或除錯本機/遠端系統。
- 使用者提到 Claude Code、Pi、OpenCode、OmniRoute、Agent、Skill 或 MCP。
- 需要 Obsidian、vault、知識庫、筆記或本機文件搜尋。
- 需要 Windows、PowerShell、symlink、junction、設定檔搬移或桌面操作。
- 需要瀏覽器、Chrome、CDP、Playwright、自動化或網頁測試。
- 需要 Docker、SSH、Cloudflare Tunnel、FRP、ESXi、GPU、ComfyUI、語音、影像、OCR、PDF、DOCX、XLSX 或其他專業工具。
- 使用者要新增、更新、搜尋、整理或驗證 `D:\OB\skills` 裡的 Skill。

不用於：

- 純閒聊、翻譯或一般知識問答，且不需要本機流程。
- 同一回合已經讀過明確匹配的 Skill，且任務範圍沒有改變。

## Procedure

### 1. 抽取搜尋關鍵字

把使用者需求轉成精簡關鍵字，包含：

- 工具、平台、專案名稱。
- 錯誤訊息或重要路徑。
- 任務動詞，例如 `install`、`configure`、`debug`、`deploy`、`search`。
- 中英文同義詞，例如 `瀏覽器/browser/chrome/cdp/playwright`、`知識庫/knowledge/obsidian/vault`、`修復/debug/troubleshoot/fix`。

### 2. 優先搜尋本機索引

`D:\OB\skills` 是正式來源；不需要先連 Obsidian MCP 才能找 Skill。依序使用：

```text
D:\OB\skills\SKILLS_INDEX.md
D:\OB\skills\_consolidation\SKILLS_CATALOG.json
D:\OB\skills\_consolidation\SKILLS_TAG_INDEX.md
D:\OB\skills\_consolidation\SKILLS_SEARCH_GUIDE.md
```

可用的唯讀搜尋方式：

```powershell
rg -n -i "<keyword>" D:/OB/skills/SKILLS_INDEX.md D:/OB/skills/_consolidation D:/OB/skills/*/SKILL.md
```

或讀取 catalog 後依 `id`、`description`、`annotation`、`category`、`tags`、`when_to_use` 搜尋。搜尋時先用工具名加任務動詞縮小範圍，不要把整個 catalog 載入上下文。

### 3. 只有需要 Obsidian app/vault 狀態時才使用 Obsidian MCP

以下情況才需要 Obsidian MCP：active file、workspace 狀態、vault link/tag 語意、Obsidian 插件搜尋，或直接對 note 進行讀寫。

- 先確認 MCP server 與工具狀態，不要假設工具名稱。
- 若 Obsidian MCP timeout 或不可用，立即回到 `D:\OB\skills` 的檔案索引，不要因此判定 Skill 不存在。

### 4. 選擇候選 Skill

按下列順序判斷：

1. `description` 或 `When to Use` 明確命中。
2. Skill 名稱與工具/專案完全匹配。
3. tags/category 與任務一致。
4. 搜尋結果多次命中同一 Skill。

通常選 1–3 個候選，不要只因名稱相似就載入大量 Skill。

### 5. 讀取完整 SKILL.md

找到候選後，必須讀取完整檔案：

```text
D:\OB\skills\<skill-id>\SKILL.md
```

不要只依索引摘要或 Skill 名稱執行。若 Skill 內引用相對路徑，路徑基準是該 Skill 所在資料夾。

### 6. 執行與回報

- 依候選 Skill 的流程處理任務。
- 若多個 Skill 有衝突，優先採用更具體、與工具/專案完全匹配者；必要時向使用者說明。
- 若沒有明確匹配，改用一般 Claude Code 流程，並簡短說明「已查過 `D:\OB\skills`，目前沒有明確匹配」。
- 若有載入 Skill，可簡短標註匹配的 Skill 名稱，不要貼出整份 Skill 內容。

## Claude Code 操作限制

- 不要把 Pi 的 `/skill-`、`skill-update`、`pi --no-skills --skill ...` 當成 Claude Code 的控制介面。
- 不要宣稱 Claude Code 有 `--skill <path>` 參數；目前的路由依賴自動發現的 `SKILL.md` 與 description 觸發。
- 不要直接停用全部 Skills 來模擬 Pi 的 `--no-skills`。若使用者要降低上下文，應先確認哪些 Skill 是 router 間接依賴，再提出精確、可逆的設定調整。
- 不要將 MCP tool 與 Skill 混為一談：MCP 是可呼叫的工具連線；Skill 是描述工作流程的 `SKILL.md`。
- 不要因為某個 Skill 的直接 usage counter 是零，就判定它沒有被 router 間接使用。
- 不要執行破壞性操作、洩漏 credentials、token 或私密資料；涉及刪除、停用、覆寫或遠端變更時，先取得使用者明確確認。

## Maintenance

若使用者要求修改或新增正式 Skill：

1. 優先檢查 `D:\OB\skills` 是否已有可改善的 Skill，避免建立重複項目。
2. 正式 Skill 必須放在 `D:\OB\skills\<lowercase-kebab-name>\SKILL.md`。
3. 修改 metadata 後，執行：
   ```text
   python D:\OB\skills\obsidian-mcp-skill-router\scripts\rebuild_skills_index.py
   ```
4. 以 `D:\OB\skills\skill-creator\scripts\quick_validate.py` 驗證新 Skill。
5. 不要修改原本給 Pi 使用的 `obsidian-mcp-skill-router`，除非使用者另外要求。

## Verification

- 確認 `D:\OB\skills\claude-obsidian-skill-router\SKILL.md` 存在且 frontmatter 合法。
- 確認 `name` 與資料夾名稱都是 `claude-obsidian-skill-router`。
- 確認索引與 catalog 能找到新 Skill。
- 確認新 Claude Code session 能發現此 Skill。
- 確認 Pi 的既有啟動方式仍保持原樣，且文件明確標示它不適用於 Claude Code。
