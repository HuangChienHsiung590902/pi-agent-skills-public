---
name: claude-cli
version: 1.0.0
description: Invoke local Claude Code CLI from command line for executing tasks, API calls, or browser automation. Use when the user wants to leverage their locally installed Claude to perform actions.
---

# Claude CLI Invoker

## Overview

本機已安裝 Claude Code CLI（`C:\Users\HCH\.local\bin\claude.exe`）。可用於在自動化流程中叫用 Claude 執行任務。

## Invocation Pattern

### 基本指令

```powershell
powershell -Command "& 'C:\Users\HCH\.local\bin\claude.exe' -p --dangerously-skip-permissions --no-session-persistence '<prompt>'"
```

### 參數說明

| 參數 | 說明 |
|------|------|
| `-p` | Pipe mode，讀取 stdin 或 prompt |
| `--dangerously-skip-permissions` | 跳過許可確認（headless 環境必要） |
| `--no-session-persistence` | 不保留對話狀態，每次獨立執行 |

## 使用情境

### 1. ECP API 自動化

讓 Claude 登入 ECP 並執行操作：

```powershell
powershell -Command "& 'C:\Users\HCH\.local\bin\claude.exe' -p --dangerously-skip-permissions --no-session-persistence '登入 http://10.145.119.234:12821/ecp/ 然後查詢帳號列表'"
```

### 2. 網頁瀏覽自動化

讓 Claude 操作瀏覽器：

```powershell
powershell -Command "& 'C:\Users\HCH\.local\bin\claude.exe' -p --dangerously-skip-permissions --no-session-persistence '開啟瀏覽器前往 https://example.com 截圖'"
```

### 3. 檔案處理

讓 Claude 處理檔案：

```powershell
powershell -Command "& 'C:\Users\HCH\.local\bin\claude.exe' -p --dangerously-skip-permissions --no-session-persistence '讀取 D:\test.csv 並回傳前10行的內容'"
```

## 重要限制

- **無 Playwright MCP**：本機 Claude 無法使用 Playwright MCP 工具，若需要瀏覽器自動化，仍需透過目前 Agent 的 Playwright 工具。
- **獨立 Session**：每次呼叫都是全新 session，無法和前一次呼叫共享狀態。
- **輸出為文字**：Claude 的回覆會以文字形式返回到 PowerShell stdout。

## 實務注意事項

1. **prompt 裡有單引號會衝突**：prompt 若包含單引號，需改用雙引號包住或先轉義。
2. **長 prompt 建議用檔案**：若 prompt 很長，可先寫入文字檔再用 `Get-Content` 傳入。
3. **編碼問題**：中文 prompt 在 Windows PowerShell 可能需指定 `-Encoding UTF8`。

## 範例：ECP 分頁查詢

```powershell
# 登入並查第2頁
$prompt = @"
登入 http://10.145.119.234:12821/ecp/
使用 Administrator / <ECP_PASSWORD>
然後查詢 Qs.Account.List 的第 2 頁帳號
"@

powershell -Command "& 'C:\Users\HCH\.local\bin\claude.exe' -p --dangerously-skip-permissions --no-session-persistence '$prompt'"
```

---

## Conformance Addendum

## When to Use
Invoke local Claude Code CLI from command line for executing tasks, API calls, or browser automation. Use when the user wants to leverage their locally installed Claude to perform actions.

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
