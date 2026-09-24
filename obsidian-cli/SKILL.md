---
name: obsidian-cli
description: Use when user wants to interact with Obsidian vault via command line, create/read/search notes, manage daily notes, tags, properties, or any Obsidian CLI operations
---

# Obsidian CLI (Official)

Obsidian 內建的官方命令列工具，透過 CLI 控制運行中的 Obsidian。

## 必要條件

- **Obsidian 1.12.4+** 已啟用 CLI
- 啟用方式：Obsidian → 設定 → General → Command line interface → Enable → Register CLI
- Obsidian 必須**處於運行狀態**（CLI 是遙控器，不是無頭模式）

## Vault 路徑

```
D:\OB
```

## CLI 位置

```
C:\Program Files\Obsidian\Obsidian.exe
```

在 bash/PowerShell 中使用完整路徑，或將 `C:\Program Files\Obsidian\` 加入 PATH。

**注意**：使用 `Obsidian.exe` 而非 `Obsidian.com`。PowerShell 輸出可能需要 redirect 到檔案才能正確顯示。範例：
```powershell
& 'C:\Program Files\Obsidian\Obsidian.exe' vault 2>&1 | Out-File -FilePath output.txt; Get-Content output.txt
```

## 命令語法

```bash
obsidian <command> [param=value] [flags]
```

## 常用命令速查

### 每日筆記
```bash
obsidian daily                                    # 開啟今日筆記
obsidian daily:append content="- [ ] 新任務"     # 附加內容
obsidian daily:yesterday                          # 昨日
obsidian daily:tomorrow                          # 明日
```

### 檔案操作
```bash
obsidian create name="Inbox/筆記名稱" content="# 標題\n內容"  # 建立筆記
obsidian read path="資料夾/檔案.md"              # 讀取
obsidian append file="檔案.md" content="\n新內容" # 附加
obsidian overwrite file="檔案.md" content="# 新內容"  # 覆寫
obsidian move from="舊.md" to="新.md"            # 移動（自動重寫 wikilinks）
obsidian delete file="不需要.md"                  # 刪除
```

### 搜尋
```bash
obsidian search query="關鍵字"                    # 全文搜尋
obsidian search:context query="關鍵字" context=3  # 含上下文
```

### 標籤
```bash
obsidian tags                                    # 列出所有標籤
obsidian tag tag="#工作"                         # 搜尋特定標籤
obsidian tags:rename old="#舊" new="#新"         # 重新命名
```

### 屬性 (Frontmatter)
```bash
obsidian properties file="檔案.md"                # 顯示屬性
obsidian property:set file="檔案.md" key="tags" value="work,test"
obsidian property:remove file="檔案.md" key="自訂"
```

### Vault 資訊
```bash
obsidian files total                             # 檔案數量
obsidian vault                                   # vault 名稱
obsidian links                                   # 所有連結
obsidian backlinks file="檔案.md"                 # 某檔案的反向連結
obsidian orphans                                # 孤兒檔案
```

### 開啟與瀏覽
```bash
obsidian open file="檔案.md"                     # 在 Obsidian 中開啟
obsidian open path="folder/file.md" newtab       # 新分頁開啟
obsidian                                       # 進入 TUI 互動模式
```

## 參數格式

- 參數：`key=value`（有空格的需加引號）
- 標幟：直接寫（如 `newtab`）
- 目標 vault：`vault="Vault名稱"`

```bash
obsidian vault="我的知識庫" search query="TODO"
obsidian open path="Inbox/靈感.md" newtab
```

## 輸出選項

| 選項 | 說明 |
|------|------|
| `--clipboard` | 複製到剪貼簿 |
| `--json` | JSON 格式輸出 |

## 限制

1. **Obsidian 必須運行** - 關閉 app 後 CLI 無法運作
2. **Desktop only** - 無法在伺服器上執行
3. **非同步操作** - 大量檔案操作建議用腳本分批執行

## 實用範例

```bash
# 建立每日任務筆記
obsidian daily:append content="- [ ] 回覆 email"

# 建立新筆記到 Inbox
obsidian create name="Inbox/想法" content="# 隨機想法\n- 第一點"

# 搜尋所有 TODO
obsidian search query="TODO"

# 移動檔案（自動更新連結）
obsidian move from="temp.md" to="Projects/工作.md"

# 列出所有 #work 標籤的檔案
obsidian tag tag="#work"

# 匯出 JSON 格式搜尋結果
obsidian search query="會議紀錄" --json
```

## 注意事項

- 使用 `file=` 適用 wikilink 解析（自動找同名檔案）
- 使用 `path=` 適用完整路徑
- 刪除預設移到垃圾桶，`permanent` 才永久刪除
- TUI 模式有自動完成和歷史記錄

---

## Conformance Addendum

## When to Use
Use when user wants to interact with Obsidian vault via command line, create/read/search notes, manage daily notes, tags, properties, or any Obsidian CLI operations

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
