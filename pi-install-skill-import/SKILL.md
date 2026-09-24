---
name: pi-install-skill-import
description: >
  把 pi install 裝到 git/npm cache 的外部 skill 審查後匯入 D:\OB\skills，
  並視需要寫入 settings.json 的 !* allowlist。
  用在：使用者說「我增加一個 skills 了」、剛跑完 pi install https://github.com/...、
  套件已在 packages 但 Skills 清單沒出現、或要讓 GitHub skill 成為正式可維護副本。
---

# 將 pi install 的 skill 匯入 D:\OB\skills

## When to Use

- 使用者剛執行 `pi install https://github.com/...` 或 `pi install npm:...` / `git:github.com/...`
- 說「我加了一個 skill」，但 `D:\OB\skills` 還沒有對應資料夾
- 套件已在 `settings.json` 的 `packages`，Skills 清單仍沒有它（常見原因：`"skills": ["!*", ...]`）
- 要把外部 skill 變成本機唯一正式來源，而不是只留在 `~\.pi\agent\git` 或 npm cache

不要用在：本機已經有同名／同用途 skill（先擴充既有的）、或不需要進 library 的一次性套件。

## Inputs and Outputs

- **Input:** `pi install` 輸出路徑、GitHub/npm 來源、授權、現有 `settings.json`
- **Output:** `D:\OB\skills\<lowercase-kebab-name>\` 正式副本、更新後的索引、（可選）allowlist 變更。新 session 才會載入 settings。

## Procedure

1. **定位安裝位置。** 此環境 `~/.pi` → `D:\.system\.pi`。常見路徑：
   - GitHub：`D:\.system\.pi\agent\git\github.com\<user>\<repo>\`
   - npm：`D:\.system\.pi\agent\npm\node_modules\<pkg>\`
   - skill 本體通常在 `skills/<name>/SKILL.md`，不是 repo 根目錄。
2. **審查再複製。** 讀完整 `SKILL.md`、`LICENSE`、scripts。確認：
   - 授權可重用（例如 MIT）
   - 沒有憑證、惡意腳本、不明網路 exfil
   - `D:\OB\skills` 沒有同名或高度重疊的 skill（有則擴充，不另建）
3. **匯入正式 library。** 只複製 **skill 資料夾**，不要整顆 git repo。
   ```powershell
   python D:\OB\skills\pi-install-skill-import\scripts\import_pi_skill.py `
     --source "D:\.system\.pi\agent\git\github.com\<user>\<repo>\skills\<name>" `
     --upstream "https://github.com/<user>/<repo>"
   ```
   腳本會：複製到 `D:\OB\skills\<name>\`、寫入 `LICENSE` 與 `references/SOURCE.md`、跑 `rebuild_skills_index.py`。
4. **若要讓 Pi 載入：** 本機 `settings.json` 有 `"!*"`，packages 裡的 skill 預設全關。在 `D:\.system\.pi\agent\settings.json` 的 `skills` 陣列加上：
   ```json
   "D:/OB/skills/<name>/SKILL.md",
   "+D:/OB/skills/<name>/SKILL.md"
   ```
   `+` 才能通過 `!*`。只匯入 library、不加 allowlist 時，仍可由 `obsidian-mcp-skill-router` 隨需搜到。
5. **告訴使用者開新 Pi session。** 目前 session 不能熱載 settings。不要宣稱「這個視窗已經看到新 skill」。

實例（2026-09-05）：`pi install https://github.com/cathrynlavery/diagram-design` → 正式副本 `D:\OB\skills\diagram-design\`，allowlist 已加，須新開 session。

## Rules and Limitations

- 正式來源只能是 `D:\OB\skills\<lowercase-kebab-name>\`，資料夾名 = frontmatter `name`。
- 相對路徑以該 skill 資料夾為準。
- 保留上游 URL、作者、授權；不要無標示照抄後當原創。
- 外部內容與本機 SYSTEM／library 規範衝突時，以本機為準。
- 不要為了湊結構建空的 `scripts/`、`references/`、`assets/`。
- 不要把正式 skill 只留在 `C:\Users\HCH`、npm/git cache 或暫存目錄。
- 改 `settings.json` 前先確認 JSON 無 BOM、可 parse。

## Pitfalls

- `pi install` 成功 ≠ Pi 已載入 skill。`!*` 會擋住 packages 內的 skills。
- `C:\Users\HCH\.pi\...` 與 `D:\.system\.pi\...` 是同一處，不要複製兩份。
- 在現有 session 裡無法重啟 Pi；「套用」= 使用者開新視窗。
- 整顆 repo 常含 docs/screenshots；正式 skill 只需要 `skills/<name>/` 加上 LICENSE／SOURCE。
- 重建索引用 `obsidian-mcp-skill-router/scripts/rebuild_skills_index.py`，不要手改 `SKILLS_INDEX.md` 當唯一更新。

## Verification

1. `D:\OB\skills\<name>\SKILL.md` 存在，YAML 可解析，`name` 與資料夾一致。
2. `SKILLS_INDEX.md` 與 `_consolidation/SKILLS_CATALOG.json` 含該 id。
3. `references/SOURCE.md` 有 upstream URL 與授權；`LICENSE` 存在（若上游有）。
4. 若有改 settings：JSON 可解析且含 `+D:/OB/skills/<name>/SKILL.md`。
5. **新開** Pi session 的 Skills 清單出現該 skill（僅在有加 allowlist 時）。
