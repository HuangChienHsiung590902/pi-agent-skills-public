---
name: gog
description: 當使用者想要透過命令列查看 Gmail、Calendar、Drive、Contacts 或其他 Google 服務時使用
---

# gog CLI - Google Workspace 命令列工具

gog 是 Google Workspace 的統一 CLI（支援 Gmail、Calendar、Drive、Contacts、Tasks、Sheets、Docs、Slides 等）。

## 設定（一次性）

```bash
# 1. 新增 OAuth 憑證
gog auth credentials "path\to\client_secret.json"

# 2. 授權帳號
gog auth add your@gmail.com

# 3. 設定預設帳號（選填）
set GOG_ACCOUNT=your@gmail.com    # Windows
export GOG_ACCOUNT=your@gmail.com  # Linux/Mac
```

## 快速參考

| 服務 | 指令 |
|---------|---------|
| **Gmail** | `gog gmail ...` |
| **Calendar** | `gog calendar ...` |
| **Drive** | `gog drive ...` |
| **Contacts** | `gog contacts ...` |
| **Tasks** | `gog tasks ...` |
| **Sheets** | `gog sheets ...` |
| **Docs** | `gog docs ...` |

## Gmail 指令

```bash
# 搜尋郵件（Gmail 查詢語法）
gog gmail messages search "is:unread" --max 10 -a your@gmail.com
gog gmail messages search "in:inbox label:unread" --max 20 -a your@gmail.com
gog gmail messages search "after:2026/01/01 before:2026/03/01" -a your@gmail.com

# 列出標籤
gog gmail labels list -a your@gmail.com

# 寄送郵件
gog gmail send --to "recipient@mail.com" --subject "Subject" --body "Body"

# 讀取郵件內容
gog gmail get <messageId> -a your@gmail.com

# 管理草稿
gog gmail drafts list -a your@gmail.com
gog gmail drafts get <draftId> -a your@gmail.com
```

## Calendar 指令

```bash
# 列出事件
gog calendar events list --max 10 -a your@gmail.com
gog calendar events list --from 2026-03-01 --to 2026-03-31 -a your@gmail.com

# 建立事件
gog calendar events create --title "Meeting" --start "2026-03-20T14:00" --end "2026-03-20T15:00" -a your@gmail.com

# 列出行事曆
gog calendar calendars list -a your@gmail.com

# 查詢空閒／忙碌時間
gog calendar freebusy --start 2026-03-20T09:00 --end 2026-03-20T18:00 -a your@gmail.com
```

## Drive 指令

```bash
# 列出檔案
gog drive ls -a your@gmail.com
gog drive ls --folder "My Drive" -a your@gmail.com

# 搜尋檔案
gog drive search "filename" -a your@gmail.com
gog drive search "mimeType='application/pdf'" -a your@gmail.com

# 上傳檔案
gog drive upload "C:\path\to\file.txt" -a your@gmail.com

# 下載檔案
gog drive download <fileId> -a your@gmail.com

# 建立資料夾
gog drive mkdir "New Folder" -a your@gmail.com
```

## Contacts 指令

```bash
# 列出聯絡人
gog contacts list -a your@gmail.com
gog contacts list --max 50 -a your@gmail.com

# 搜尋聯絡人
gog contacts search "John" -a your@gmail.com
```

## Tasks 指令

```bash
# 列出任務清單
gog tasks lists list -a your@gmail.com

# 列出清單中的任務
gog tasks tasks list <listId> -a your@gmail.com

# 建立任務
gog tasks tasks create --list <listId> --title "Buy milk" -a your@gmail.com

# 完成任務
gog tasks tasks complete <taskId> -a your@gmail.com
```

## Sheets 指令

```bash
# 列出試算表
gog sheets list -a your@gmail.com

# 讀取試算表資料
gog sheets read <spreadsheetId> --sheet "Sheet1" -a your@gmail.com

# 寫入資料
gog sheets write <spreadsheetId> --sheet "Sheet1" --data '[["A1","B1"],["A2","B2"]]' -a your@gmail.com
```

## 常用旗標

| 旗標 | 說明 |
|------|-------------|
| `-a, --account` | 指定帳號 email |
| `-j, --json` | 輸出 JSON 格式（適合腳本使用） |
| `-p, --plain` | 輸出純文字格式（TSV） |
| `-y, --force` | 跳過確認提示 |
| `-v, --verbose` | 啟用詳細記錄 |

## JSON 輸出

使用 `-j` 或 `--json` 取得機器可讀格式：

```bash
gog gmail messages search "is:unread" --max 5 -j -a your@gmail.com | jq '.messages[].subject'
```

## 多帳號管理

```bash
# 新增另一個帳號
gog auth add work@gmail.com

# 使用特定帳號
gog gmail messages search "is:unread" -a hch590902@gmail.com
gog calendar events list -a work@gmail.com
```

---

## Conformance Addendum

## When to Use
當使用者想要透過命令列查看 Gmail、Calendar、Drive、Contacts 或其他 Google 服務時使用

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
