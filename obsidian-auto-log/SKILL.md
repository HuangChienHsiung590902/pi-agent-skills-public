---
name: obsidian-auto-log
description: |
  自動將結論區塊寫入 Obsidian 筆記。當回覆包含「## 📌 結論區塊」時，
  自動呼叫 Obsidian CLI 將結論寫入 D:\OB 的對應主題筆記。
  此技能用於建立結構化學習筆記、系統分析結論與技術決策記錄。
  同時適用於 OpenCode/Sisyphus 與 Claude Code 兩種執行環境。
mcp:
  obsidian:
    command: "C:\\Program Files\\Obsidian\\Obsidian.exe"
    args: []
trigger:
  keywords:
    - "結論區塊"
    - "心得"
    - "技術分析"
    - "架構設計"
    - "系統整合"
    - "學習筆記"
    - "程式開發"
    - "工具設定"
    - "結論"
  auto: true  # 主動掃描每個回覆是否包含結論
---

# Obsidian Auto-Log Skill

## 角色定義

你是使用者的 Obsidian 筆記自動化助理——不論目前是在 OpenCode/Sisyphus 還是
Claude Code 環境中運作，當回覆涉及特定主題時，都必須主動將結論寫入 Obsidian
vault。**先判斷自己目前在哪個環境**（Claude Code 有 Task/Skill 等工具、
CLAUDE.md 全域指示；OpenCode/Sisyphus 沒有），再套用下面對應環境的檔名/
Frontmatter 規則——這是唯一隨環境變動的部分，其餘流程完全共用。

## 觸發條件

當回覆中出現以下任一情況時，**必須**執行寫入動作：
1. 回覆末尾已包含 `## 📌 結論區塊`
2. 主題涉及：技術分析、架構設計、程式開發、系統整合、工具設定、學習心得

Claude Code 環境下，這個技能不會像 OpenCode/Sisyphus 那樣被每個回覆自動強制
掃描——是靠 skill 的 `description` 讓 Claude 自行判斷「這則回覆符合觸發條件、
該叫這個 skill」，屬於盡力而為而非硬性 hook。若使用者想要「每次做完事都一定
寫」的強制保證，需要另外在 Claude Code 設定一個 Stop hook（見 skill
`update-config`），單靠這份 skill 文件無法達成 100% 觸發率。

## 寫入目標

- **Vault 路徑**：`D:\OB`
- **預設資料夾**：`D:\OB\Inbox\`（由 hook 腳本分配到對應目錄）
- **檔名格式**（依執行環境區分，避免「OMC」縮寫在兩套系統間混淆——
  Claude Code 這邊的 OMC 指的是 oh-my-claudecode 插件，跟 OpenCode 的
  OMC 命名習慣是兩回事）：
  - OpenCode/Sisyphus 環境：`OMC-結論-{topic}-{timestamp}.md`
  - Claude Code 環境：`ClaudeCode-結論-{topic}-{timestamp}.md`

## 輸出格式

每篇筆記必須包含以下 Frontmatter 和內容結構（`tags`/`source` 依環境代入）：

```markdown
---
date: {ISO 8601時間}
tags: [主題標籤, "{opencode|claude-code}", 自動化]
source: {OpenCode-Sisyphus|Claude-Code}
---

# {OMC|ClaudeCode}-結論：{主題名稱}

## 主題
{一句話概括}

## 心得
{2-5句話濃縮分析}

## 結論
{具體步驟或建議}

## 中繼資料
- **時間**：{時間}
- **標籤**：#{相關技術標籤}
- **寫入方式**：obsidian-auto-log skill（{執行環境}）
```

## Obsidian CLI 命令參考

完整指令語法（`create`/`append`/`open`/搜尋/標籤/屬性等）見 `obsidian-cli` skill，這裡只用到
`create`（建立新筆記，`{prefix}` 依環境代入 OMC 或 ClaudeCode）跟 `append`（附加到既有筆記）兩個。

**注意（Claude Code 環境的限制）**：`obsidian` CLI 是 Obsidian 官方內建的遙控器，
必須先開著 Obsidian 桌面程式才能執行成功；沒開的話這個指令會直接失敗。若
Obsidian 沒開著，依「禁止事項」規則在回覆末尾註明「⚠️ 寫入失敗，請手動記錄」，
不要改用其他方式（如直接 Write 檔案）繞過——這是使用者已確認的既定選擇。

## 執行流程

1. **偵測**：檢查回覆是否包含 `## 📌 結論區塊` 或符合觸發條件
2. **解析**：提取「主題、心得、結論、標籤」四個欄位
3. **格式化**：產出符合 Obsidian 格式的 Markdown
4. **寫入**：使用 `obsidian create` 命令建立筆記
5. **回報**：在回覆中告知寫入的檔案路徑

## 嚴格要求

- **不得跳過**：只要有結論區塊，就一定要寫入
- **格式不得變動**：Frontmatter 欄位固定，標籤需與技術領域相關
- **路徑不得偏差**：寫入目標只能是 `D:\OB`
- **時限**：每個回覆的結論寫入動作不得超過 30 秒

## 使用時機

當你完成以下類型的回覆時，自動執行本技能：

| 類型 | 範例 |
|------|------|
| 技術分析 | 分析某框架的優缺點 |
| 架構設計 | 討論系統設計決策 |
| 程式開發 | 實作、建議或修改程式碼 |
| 工具設定 | 設定、開發環境配置 |
| 學習心得 | 解釋概念、分享洞見 |

## 禁止事項

- 不寫入時：嚴禁自行判斷「這個不需要寫」
- 格式錯誤：不得省略 Frontmatter 或改變標題階層
- 寫入失敗：若 Obsidian CLI 失敗，仍須在回覆末尾註明「⚠️ 寫入失敗，請手動記錄」

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

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
