# Skills Search Guide

> 根目錄：`D:/OB/skills`；catalog：`_consolidation/SKILLS_CATALOG.json`。索引時間以 catalog 的 `generated_at` 為準。

## 查詢流程

1. 先看根目錄 `SKILLS_INDEX.md` 的「快速查找」與分類。
2. 若知道工具/平台/任務關鍵字，查 `_consolidation/SKILLS_TAG_INDEX.md`。
3. 若要程式化查詢，讀 `_consolidation/SKILLS_CATALOG.json`，依 `id`、`description`、`annotation`、`category`、`tags`、`when_to_use` 搜尋。
4. 找到候選 skill 後，讀 `D:/OB/skills/<skill-id>/SKILL.md`；相對路徑以該 skill 目錄為基準。
5. 若沒有合適 skill，先全文搜尋再決定是否建立新 skill，避免重複。

## 推薦搜尋指令

```powershell
# 關鍵字全文搜尋
rg -n -i "<keyword>" D:/OB/skills/SKILLS_INDEX.md D:/OB/skills/_consolidation D:/OB/skills/*/SKILL.md

# PowerShell 查 JSON catalog
$cat = Get-Content -Raw D:/OB/skills/_consolidation/SKILLS_CATALOG.json | ConvertFrom-Json
$cat.skills | Where-Object { ($_.id + ' ' + $_.description + ' ' + $_.annotation + ' ' + ($_.tags -join ' ')) -match '<keyword>' } | Select-Object id,category,description

# jq 查 catalog（Git Bash）
jq -r '.skills[] | select((.id+.description+.annotation+(.tags|join(" ")))|test("<keyword>"; "i")) | [.id,.category,(.tags|join(",")),.description] | @tsv' D:/OB/skills/_consolidation/SKILLS_CATALOG.json
```

## 常用關鍵字

- agent/模型：`pi`, `claude`, `opencode`, `omniroute`, `mcp`, `llm`, `qwythos`, `qwen`, `vllm`
- 瀏覽器/自動化：`browser`, `chrome`, `cdp`, `playwright`, `obsidian`
- 本機/Windows：`windows`, `powershell`, `symlink`, `junction`, `desktop`, `hardware`
- 基礎設施：`docker`, `ssh`, `plink`, `cloudflare`, `frp`, `esxi`, `server`, `gcp`
- 媒體/AI：`comfyui`, `video`, `audio`, `whisper`, `vision`, `gpu`, `ocr`
- 專案域：`ecp`, `aipower`, `chainsea`, `cbm`, `line`, `vrs`
- 文件/知識庫：`docx`, `pdf`, `xlsx`, `pandoc`, `anytxt`, `knowledge`, `search`, `obsidian`

## Agent 使用規則

- 不要只憑 skill 名稱猜流程；找到候選後一定讀該 `SKILL.md`。
- 若任務涉及本機設定，優先相信 `D:/OB/skills` 與 `D:/` 實際路徑；`C:/Users/HCH` 可能是 symlink/junction 相容層。
- 修改 Skill 前先 `git status`；刪除或搬移既有 Skill 必須先取得使用者確認並建立備份。
- 對長期可重複的流程，新增 `D:/OB/skills/<slug>/SKILL.md`，再執行 `D:/OB/skills/obsidian-mcp-skill-router/scripts/rebuild_skills_index.py` 重新產生索引。
