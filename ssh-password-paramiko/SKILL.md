---
name: ssh-password-paramiko
description: Use when the user provides an SSH target with an inline password like `ssh user@host pwd:...`, or when Claude Code cannot answer an SSH password prompt non-interactively. Connect with Python Paramiko, not raw OpenSSH/sshpass; first verify hostname and detect remote OS before running OS-specific commands.
triggers:
  - "pwd:"
  - "SSH password"
  - "ssh 密碼"
  - "ssh user@host pwd"
argument-hint: "ssh <user>@<host> pwd:<password> [command]"
---

# SSH 密碼登入：Paramiko 方法

## 何時使用

使用者給這種格式時：

```text
ssh administrator@192.168.100.110 pwd:<USB_ACCOUNT_PASSWORD>
```

不要把它當成真正 shell 指令直接丟給 `ssh`。OpenSSH 沒有 `pwd:` 這個語法，而且 Claude Code 的非互動工具不能穩定回答密碼提示。

**直接用 Python `paramiko` 把 password 傳給 SSH 協定。** 這就是本機已驗證可行的方法；`administrator@192.168.100.110` 曾用此法一次連上。

## 最小流程

1. 從使用者字串解析：`user`、`host`、`password`。
2. 不要把密碼寫進檔案；用環境變數傳入一次性 Python 腳本。
3. 先跑 `hostname` 確認連線。
4. 再判斷遠端 OS：
   - 先試 `uname -s 2>/dev/null`
   - 失敗再試 `cmd /c ver 2>NUL`
5. 確定 Windows/Linux 後才跑對應命令。

## PowerShell 呼叫範例

```powershell
$env:SSH_HOST='192.168.100.110'
$env:SSH_USER='administrator'
$env:SSH_PASS='<password>'
$env:SSH_CMD='hostname'
python "$HOME\.claude\skills\ssh-password-paramiko\scripts/ssh_password_paramiko.py"
```

多個命令用換行分隔：

```powershell
$env:SSH_CMD=@'
hostname
cmd /c ver
cd
'@
python "$HOME\.claude\skills\ssh-password-paramiko\scripts/ssh_password_paramiko.py"
```

## 注意

- Windows 遠端的 `stderr` 可能因編碼顯示亂碼；只要 exit code 與 stdout 正常，不要誤判成連線失敗。
- 需要互動式 shell 或設定免密碼登入時，改用既有 skill：`ssh-passwordless-windows`。
- 如果 `paramiko` 不存在，先檢查是否可用 `plink -ssh -batch -pw <password> user@host "command"`；不要優先用 `sshpass`，Windows 上常不穩。

---

## Conformance Addendum

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
