---
name: skills-library-conformance-maintenance
description: >
  當使用者要求「整理 skills」、「檢查所有 skills 是否合規」、「修復 Skill library」、
  「清理重複或失效 skills」、「更新 skills 索引」、「稽核 D:\OB\skills」、
  「補標題讓稽核變綠」或問「缺章節還能用嗎」時使用。
  對正式 Skill library 執行備份、結構與 YAML 稽核、必要章節標題修復、
  腳本與資源整理、索引重建及最終驗證；刪除、搬移或大量改寫前必須先取得使用者確認。
---

# Skills Library Conformance Maintenance

## When to Use

在下列情況使用本 Skill：

- 使用者要求整理、清理、健檢或稽核 `D:\OB\skills`。
- 新增、刪除、搬移或批次修改 Skills 後，需要確認 library 是否仍合規。
- `SKILLS_INDEX.md`、`_consolidation` catalog 與實際資料夾數量不一致。
- 發現失效 symlink、巢狀 `SKILL.md`、空 Skill 目錄、壞掉的 YAML frontmatter。
- 腳本散落在 Skill 根目錄，或存在不必要的空 `scripts/`、`references/`、`assets/`。
- 要移除整套已廢棄的 Skills，例如舊 framework 匯入的巢狀 Skill collection。
- 使用者要求「補標題讓稽核變綠」、處理 `missing required section`，或問缺章節的 Skill 能不能用。

本 Skill 管理整個 library 的合規與維護；單純建立一個新 Skill 時仍應先做名稱與 trigger 重疊檢查，再依本機 Skill Creator 格式建立。

## Inputs and Outputs

### Inputs

- 正式 Skill library，預設固定為 `D:\OB\skills`。
- 使用者要保留、移除、合併或修復的範圍。
- 是否授權刪除、搬移、批次改寫等破壞性操作。
- 必要時，指定備份輸出位置；預設為 `D:\OB\skill-backups`。

### Outputs

- 刪除或大量修改前的可還原備份。
- 合規稽核報告，列出 error、warning 與受影響路徑。
- 經使用者授權後完成的安全修復。
- 更新後的 `SKILLS_INDEX.md` 與 `_consolidation` 索引。
- 最終驗證結果，包括 Skill 數量、YAML、名稱、章節、索引與資源狀態。

## Canonical Standard

每個正式 Skill 必須符合：

1. 正式來源位於 `D:\OB\skills\<lowercase-kebab-name>\`。
2. 每個 Skill 是獨立資料夾，根目錄必須有 `SKILL.md`。
3. 資料夾名稱只能使用小寫英數與 hyphen。
4. YAML frontmatter 至少包含：
   - `name`：與資料夾名稱完全一致。
   - `description`：具 trigger 語意，清楚說明何時載入。
5. `SKILL.md` 必須涵蓋六類必要章節。稽核認的是 `##` 標題名稱，不是 loader 拒載；標題對照見 `references/required-headings.md`。
6. 確定性自動化腳本放在 `scripts/`。
7. 模板、規範、API 摘要與領域資料放在 `references/`。
8. 靜態輸入、圖片、成品模板等放在 `assets/`。
9. 不建立空的 `scripts/`、`references/`、`assets/`。
10. 相對路徑以該 Skill 資料夾為基準。
11. 正式 Skill 不得依賴 agent cache、套件 cache 或 library 外部 symlink 作為唯一來源。
12. 修改完成後必須同步根索引與 `_consolidation`。

不強制加入特定外部 framework 的 `version`、`triggers`、`tools`、`mutating`、`Contract` 或 `Phases` 欄位；除非本機正式規範日後明確要求。

## Procedure

### Phase 1：確認來源與目前狀態

1. 確認 `D:\OB\skills` 是實際來源，不要只檢查 `C:\Users\HCH` 相容路徑。
2. 執行：
   ```powershell
   git -C D:\OB\skills status --short
   python D:\OB\skills\skills-library-conformance-maintenance\scripts\audit_skills_library.py
   ```
3. 搜尋名稱、用途與 trigger 重疊，避免把既有能力再建立一份。
4. 將問題分成：
   - 可安全自動修復
   - 需要人工判斷
   - 破壞性操作，必須取得確認

### Phase 2：破壞性操作前備份

刪除 Skill、搬移大量檔案或批次改寫前：

```powershell
pwsh -File D:\OB\skills\skills-library-conformance-maintenance\scripts\backup-skills-library.ps1
```

備份完成後必須確認：

- 壓縮檔存在且大小大於零。
- 壓縮檔可以列出內容。
- 回報實際備份路徑。

備份不能取代刪除確認。即使有備份，仍必須先取得使用者對刪除範圍的明確同意。

### Phase 3：修復結構

依序處理：

1. 修正無法解析的 YAML，長 description 優先使用：
   ```yaml
   description: >
     Trigger-oriented description with quotes and colons: safely folded here.
   ```
2. 修正資料夾與 frontmatter `name` 不一致。
3. 空目錄先判斷用途：
   - 是廢棄 Skill：取得確認後刪除。
   - 是正式 Skill：補齊 `SKILL.md`。
   - 是資料集合：整理成正式 Skill，或移出 library 頂層。
4. 巢狀 Skill 必須判斷：
   - 匯入為正式頂層 Skill；或
   - 視為父 Skill 的 reference；或
   - 經確認後移除。
5. 外部 symlink 必須確認目標與正式來源；不可讓 library 外部位置成為唯一可維護版本。
6. 將可執行腳本移入 `scripts/`，並同步修正 `SKILL.md` 與 references 中的相對路徑。
7. 刪除空的選用資源目錄。
8. 清理 `__pycache__`、`*.pyc` 等可重建產物；`.venv`、browser profile、output、build cache 可能正被服務使用，未確認前不可刪除。
   本機已知且有用途的 runtime 目錄（`aipower-docker-local/.build-cache`、`audiocpp-realtime-web/web/.venv`、`notebooklm/.venv`、`notebooklm/data/browser_state`）由稽核工具列入 allowlist；不要為了消除 warning 擅自刪除。
9. 補齊缺少的必要章節時，依 `references/required-headings.md`：
   - 先說明：缺的是標題名稱，Skill 通常仍能被 router 找到並整份讀取。
   - 能改現有對等標題就改，避免重複章節。
   - 沒有對等標題才從原文抽出一小段追加；禁止通用模板覆蓋領域知識。
   - 只改內文標題、frontmatter 沒變時，跳過索引重建，但仍須重跑稽核。
   - `scripts/repair-required-sections.py` 只處理它 allowlist 裡的舊檔，不要為了新的缺漏去擴那份清單。

### Phase 4：驗證腳本與相對路徑

1. Python 腳本執行語法檢查：
   ```powershell
   python -m py_compile <script.py>
   ```
2. PowerShell 腳本至少用 parser 驗證語法；可能改動系統的腳本不可為了測試而直接執行。
3. 檢查 `SKILL.md` 內相對連結與 `scripts/`、`references/`、`assets/` 引用是否存在。
4. 第三方內容需保留來源、作者與授權資訊，並先審查安全性。

### Phase 5：重建索引

frontmatter 的 `name`／`description` 沒變、只改內文章節時，跳過本階段。否則執行正式索引工具：

```powershell
python D:\OB\skills\obsidian-mcp-skill-router\scripts\rebuild_skills_index.py
```

應同步更新：

- `D:\OB\skills\SKILLS_INDEX.md`
- `D:\OB\skills\_consolidation\SKILLS_CATALOG.json`
- `D:\OB\skills\_consolidation\SKILLS_BY_CATEGORY.md`
- `D:\OB\skills\_consolidation\SKILLS_TAG_INDEX.md`
- `D:\OB\skills\_consolidation\all-skill-md-paths.txt`
- `D:\OB\skills\_consolidation\all-skill-md-paths.csv`

### Phase 6：最終驗證

再次執行：

```powershell
python D:\OB\skills\skills-library-conformance-maintenance\scripts\repair-required-sections.py
python D:\OB\skills\obsidian-mcp-skill-router\scripts\rebuild_skills_index.py
python D:\OB\skills\skills-library-conformance-maintenance\scripts\audit_skills_library.py
```

只有在輸出 `errors: []` 且索引數量一致時，才能宣稱整理完成。Warning 必須逐項說明為何保留，例如仍在使用的 `.venv`。

## Rules and Limitations

- 刪除、覆蓋、批次搬移前必須提醒風險並取得確認。
- 修改前先讀取相關檔案，不憑名稱猜內容。
- 不要以批次模板覆蓋 Skill 原有的專業流程。
- 不要把外部 framework 的專用 schema 強加到全部本機 Skills。
- 不要因為看到 `.venv`、output 或 cache 就直接刪除；先判斷是否為正在使用的執行環境。
- 不要執行具有部署、刪除、資料庫修改或遠端控制副作用的腳本來做語法驗證。
- 稽核腳本預設唯讀；自動修復與刪除必須是另一個明確步驟。
- Git working tree 有既有未提交變更時，不可把所有差異都當成本次工作造成。

## Pitfalls

- Git Bash 內直接執行 PowerShell cmdlet 會失敗；Windows PowerShell 操作使用 `pwsh -File` 或明確 PowerShell command。
- YAML 單行 description 中含未引用的 `: ` 可能造成 parser error；長文字用 `>`。
- `Path.is_dir()` 可能跟隨 symlink，因此稽核時要先單獨判斷 symlink。
- Markdown link regex 可能把一般文字誤判為路徑；不存在連結應人工確認後才修改。
- 移動 sibling scripts 後，腳本內部相對 import 或 `$PSScriptRoot` 行為可能改變，必須重新驗證。
- 索引重建成功不代表 Skill 內容完整；仍需跑完整 conformance audit。
- `missing required section` 只代表 `##` 標題對不上 regex，不代表 Skill 壞掉或 Pi 不能載入。
- `標準維護流程`、`Safety`、`Known Pitfalls`、`Troubleshooting` 這類近義標題不會過稽核，優先改名而不是再貼一份。
- 備份若排除 `.venv` 或 cache，還原時不會包含這些可重建或執行環境資料，回報時要說明。

## Verification

完成條件：

1. 所有頂層正式 Skill 都有 `SKILL.md`。
2. 所有資料夾與 `name` 都符合 lowercase kebab-case 且完全一致。
3. 所有 YAML frontmatter 均可解析，且有非空 trigger-oriented `description`。
4. 必要章節完整。
5. Skill 根目錄沒有散落的可執行腳本。
6. 沒有空的 `scripts/`、`references/`、`assets/`。
7. 沒有未處理的失效 symlink 或非預期巢狀 `SKILL.md`。
8. Catalog 的 `skill_count`、entries 與實際頂層 Skills 數量一致。
9. 索引內沒有已移除 Skill 的 stale entry。
10. 稽核腳本輸出 `errors: []`；所有 warning 都有保留理由。
11. 回報備份、實際修改、驗證指令與結果。
