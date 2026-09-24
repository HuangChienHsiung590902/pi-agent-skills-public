---
name: vault-indexer
description: 為 Obsidian vault 自動產生與維護 SKILL.md 格式的快速索引。當使用者提到「產生索引」、「更新 vault 索引」、「掃描 vault」、「整理 vault 索引」，或每次 session 結束前有新筆記寫入 vault 時主動觸發。
---

# Vault Indexer

自動為 Obsidian vault 產生分類索引，格式類似 SKILL.md，供 AI Agent 快速查找文件。

## 核心邏輯

使用 PowerShell 腳本掃描 vault，自動分類並寫入 `INDEX.md`。

**工作流程：**

```
1. 掃描 vault 根目錄 + 所有子目錄（排除 .obsidian/、DOCS/）
2. 讀取每個 .md 檔的 frontmatter title（若有）與第一行 H1
3. 根據副檔名與路徑自動分類到類別目錄
4. 產出 SKILL.md 格式的 INDEX.md
5. 更新 Inbox/鉤子自動寫入.md（如果有這個檔案的話）
```

## 產出格式

```markdown
# {VAULT_NAME} Vault 快速索引

> 本索引供 AI Agent 快速查找文件使用。格式參考 SKILL.md。
> 最後更新：{DATE}

---

## 📁 類別索引

### 🤖 AI / Dev Agent 開發 (`AI-DevAgent/`)

| 檔名 | 一句話描述 | 關鍵鍵字 | 適合情境 |
|------|-----------|---------|---------|
| `xxx.md` | ... | keyword1, keyword2 | ... |

---

## 🔍 快速查閱捷徑

| 需求 | 前往 |
|------|------|
| ... | ... |
```

## 使用方式

### 手動觸發

```markdown
使用者：幫我更新 vault 索引
```

### 自動觸發（prompt_append）

在 `oh-my-opencode.json` 的 sisyphus prompt_append 加入：

```json
"prompt_append": "\n\n每次 session 結束前，檢查是否有新寫入 vault 的筆記。如果有，主動詢問使用者是否要更新 vault INDEX.md 索引。\n更新時使用 vault-indexer skill：scripts/scan-vault-index.ps1 -VaultPath 'D:\\OB' -OutPath 'D:\\OB\\INDEX.md'\n"
```

## 執行命令

```powershell
# 基本用法（使用預設路徑 D:\OB）
.\scripts/scan-vault-index.ps1

# 指定 vault 路徑
.\scripts/scan-vault-index.ps1 -VaultPath 'D:\Projects\MyVault'

# 指定輸出路徑
.\scripts/scan-vault-index.ps1 -VaultPath 'D:\OB' -OutPath 'D:\OB\INDEX.md'

# 完整參數
.\scripts/scan-vault-index.ps1 -VaultPath 'D:\OB' -OutPath 'D:\OB\INDEX.md' -VaultName 'OB'
```

## 目錄自動分類規則

| 偵測關鍵詞 | 類別名稱 |
|-----------|---------|
| `Dev Agent` / `LangGraph` / `Ollama` | AI-DevAgent |
| `ESP32` / `BLE` | ESP32 |
| `OpenCode` / `OMC` / `Skill` / `SKILL.md` | OpenCode |
| `Rime` / `注音` / `小狼毫` / `bopomofo` | Rime |
| `GenieACS` / `SIP` / `TR-069` / `ACS` | 網管 |
| `LINE` / `LINE-Bot` | LINE-Bot |
| `Pandoc` / `SSH` / `Chrome` / `Arduino` / `VRS` / `gog` | 工具 |
| `CRM` / `qbicrm` / `同事` | CRM |

未被分類的檔案統一放進 `工具/` 之外的 `其他/` 區塊。

## 腳本輸出範例

```
=== Vault Indexer ===
Vault: D:\OB
Output: D:\OB\INDEX.md
Files found: 36
Categories: 8
Completed: 2026/4/18 下午 03:50
```

## 依賴

- PowerShell 5.1+
- 讀寫檔案權限

## 參考資源

- 完整腳本，請參閱 [scripts/scan-vault-index.ps1](scripts/scan-vault-index.ps1)

---

## Conformance Addendum

## When to Use
為 Obsidian vault 自動產生與維護 SKILL.md 格式的快速索引。當使用者提到「產生索引」、「更新 vault 索引」、「掃描 vault」、「整理 vault 索引」，或每次 session 結束前有新筆記寫入 vault 時主動觸發。

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
