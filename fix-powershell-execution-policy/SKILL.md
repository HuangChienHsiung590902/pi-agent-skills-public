---
name: "fix-powershell-execution-policy"
description: "修復 Windows PowerShell 因執行原則（Execution Policy）擋住 .ps1 腳本（例如 pi、npm 全域指令）導致「running scripts is disabled on this system」錯誤"
version: 1
created: "2026-08-14"
updated: "2026-08-14"
---
## When to Use
當使用者在 Windows PowerShell 執行某個指令（例如 pi、npm 安裝的全域指令，或任何 .ps1 腳本包裝的 CLI 工具）出現錯誤訊息：「File ... cannot be loaded because running scripts is disabled on this system」或「UnauthorizedAccess / PSSecurityException」時使用。常見於使用者剛安裝 npm 全域套件（如 pi coding agent）後第一次執行指令。

## Procedure
1. 確認錯誤訊息包含 'running scripts is disabled on this system' 或 'PSSecurityException'，判斷是 PowerShell 執行原則（Execution Policy）問題，而非該工具本身安裝失敗
2. 告知使用者這是本機 Windows 安全設定問題，與遠端伺服器/SSH/其他服務設定無關
3. 請使用者在自己的 PowerShell 視窗（非透過 Bash 工具，因為 Bash 工具無法真正操作使用者本機互動式 PowerShell 環境）執行：Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
4. 說明若跳出確認提示，輸入 Y 或 A 確認執行
5. 提供一次性繞過方案作為替代（不改變全域設定）：powershell -ExecutionPolicy Bypass -File "<script路徑>.ps1"，適合使用者不想更改原則設定時使用
6. 建議設定完成後用 Get-ExecutionPolicy -List 確認 CurrentUser scope 已變成 RemoteSigned
7. 請使用者重新執行原本指令（例如 pi）確認可正常啟動

## Pitfalls
- -Scope CurrentUser 只影響該使用者帳號，不需要系統管理員權限，也不會動到系統全域（LocalMachine）原則，比 Set-ExecutionPolicy 不加 -Scope 更安全，優先使用此寫法
- 不要建議 Unrestricted（完全不驗證任何腳本簽章），RemoteSigned 已足夠讓本機產生的 .ps1（如 npm 全域安裝產生的 pi.ps1）正常執行，同時仍要求從網路下載的腳本要有簽章，較安全
- 這個操作必須在使用者自己的互動式 PowerShell 視窗執行，AI agent 端的 Bash 工具（Git Bash/WSL）無法代為執行，因為那是不同的 shell 環境，且執行原則是針對使用者的 PowerShell session/機碼設定
- 若使用者環境有透過群組原則（GPO）鎖定 ExecutionPolicy，CurrentUser scope 設定可能被覆蓋而不生效，此時需要請 IT 或系統管理員協助，或改用一次性 Bypass 繞過方案

## Verification
1. Get-ExecutionPolicy -List 顯示 CurrentUser 該列為 RemoteSigned（或比 Restricted 寬鬆的原則）
2. 使用者重新執行原本被擋的指令（例如 pi）能正常啟動，不再出現 UnauthorizedAccess / PSSecurityException 錯誤

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
