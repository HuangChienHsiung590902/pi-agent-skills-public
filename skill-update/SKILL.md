---
name: skill-update
description: >-
  當使用者輸入 /skill-update 或 skill-update 時，把本回合過程與結果更新到 D:\OB\skills 裡最相關的現有 skill，不要新建重複的。先出執行摘要，等 y 再改檔。
---

# skill-update

把剛才的過程與結果寫進現有 skills。

## When to Use

- 使用者輸入 `/skill-update` 或 `skill-update`

## Procedure

1. 搜 `D:\OB\skills`，找最相關的現有 skill。
2. 過程寫 Procedure，驗證寫 Verification，踩坑寫 Pitfalls。
3. 只改現有 skill；沒有對應的才建議新建，等 y。
4. Spark 給 4B 的 `agents.md`／`spark-skills` 保持短；長流程寫 `D:\OB\skills`。
5. 先出執行摘要，等 **y** 再改檔。
6. 改完更新 `SKILLS_INDEX.md`（必要時跑 `obsidian-mcp-skill-router/scripts/rebuild_skills_index.py`）。

## Pitfalls

- 這是一般 skill 指令。若斜線選單看不到，確認 `settings.json` 的 `skills` 有載入本 `SKILL.md`，然後重開 Pi。
- 不要把整段對話塞進 4B 的 `spark-skills`。

## Verification

1. 資料夾名與 frontmatter `name` 都是 `skill-update`。
2. Pi `settings.json` 的 `skills` 含 `D:/OB/skills/skill-update/SKILL.md`。

## Inputs and Outputs

### Inputs

- 本回合已完成的流程、實測結果、錯誤與使用者確認。
- `D:/OB/skills` 中最相關的現有 Skill；只更新既有 Skill，不重複建立。

### Outputs

- 寫入現有 Skill 的 Procedure、Pitfalls、Verification 更新。
- 必要時同步更新 `SKILLS_INDEX.md` 與 consolidation 索引，並回報實際檔案與驗證結果。

## Rules and Limitations

- 先出執行摘要，等待使用者輸入 `y` 後才修改 Skill 檔案。
- 修改前先讀取現有內容；不要用通用模板覆蓋領域知識，也不要把整段對話塞進短 skills。
- 沒有對應的現有 Skill 時，只提出新 Skill 建議，不要自行建立重複項目。
- 不寫入 credential、token 或無法重現的未驗證結論。
