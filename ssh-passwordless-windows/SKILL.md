---
name: ssh-passwordless-windows
description: Use when setting up passwordless SSH (public key) login to a Windows OpenSSH server from this machine, or when public-key auth to a Windows box is silently rejected despite correct ACLs and file content — covers "Permission denied (publickey,password,keyboard-interactive)", sshpass failing on Windows, and the administrators_authorized_keys trap for accounts in the local Administrators group.
---

# Windows OpenSSH 免密碼登入設定

幫任何 Windows OpenSSH 帳號（`OpenSSH_for_Windows_*`）建立公鑰免密碼登入。

## 直接跑腳本

```powershell
& "$HOME\.claude\skills\ssh-passwordless-windows\scripts/setup-passwordless-ssh.ps1" -TargetHost <ip> -TargetUser <user> -Password <password>
```

腳本會自動：本機沒有 ed25519 金鑰就產生一把 → 用 plink 測密碼登入 → 判斷該帳號是否為本機
Administrators 群組成員 → 寫入正確的 authorized_keys 檔案並設好 ACL → 用本機 `ssh` 驗證免密碼登入。
可重複執行，不會寫入重複的金鑰行。

跑完後直接 `ssh user@host` 就不會再要求密碼。

## 為什麼不能用 sshpass

`D:\BIN\sshpass.exe`（或任何 Windows 版 sshpass）在這台機器上會讓 OpenSSH 的密碼認證直接被拒絕
（`Permission denied`），無論用 `-p`、stdin、或 verbose 都一樣 —— 不是密碼或跳脫字元問題，就是這個
Windows port 跟 OpenSSH 的密碼 prompt 機制對不上。**改用 PuTTY 的 `plink.exe`**：

```powershell
plink -ssh -batch -pw <password> user@host "command"
```

`plink` 找不到時：`Get-Command plink` 常常撲空（不在這個 PowerShell session 的 PATH），但
`C:\Program Files\PuTTY\plink.exe` 通常都在，腳本裡已經有 fallback。真的沒裝就
`winget install PuTTY.PuTTY`。

第一次連線 plink 會跳 host key 確認 prompt，`-batch` 模式下沒有 TTY 會直接失敗。解法：先用
`"y`n" | plink -ssh -pw ...` 跑一次隨便的指令把 host key 存進快取，之後才能用 `-batch`。

## 核心陷阱：Administrators 群組帳號要用 administrators_authorized_keys

**症狀**：金鑰內容確認正確（hex dump 沒有多餘字元）、ACL 也只有 SYSTEM/Administrators/本人、
owner 也改成該使用者了，`ssh -v` 卻顯示 offer 完 public key 後立刻被拒絕、完全沒進 challenge
階段，OpenSSH event log（`Get-WinEvent -LogName "OpenSSH/Operational"`）裡連一筆 pubkey 相關記錄
都沒有（這是正常的 —— OpenSSH 對 query 階段就被拒的 key 不會記 log，所以「log 是空的」不代表沒發生
過嘗試）。

**根因**：Win32-OpenSSH 的 sshd_config 預設帶這個規則：

```
Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

只要連線帳號是本機 **Administrators 群組成員**（不只內建的 `Administrator`，任何被加進這個群組的
帳號都算，用 `net localgroup administrators` 查），sshd 就完全無視 `~/.ssh/authorized_keys`，
只認 `C:\ProgramData\ssh\administrators_authorized_keys`。這個檔案還要求嚴格的 ACL —— 只能有
`SYSTEM:F` 和 `Administrators:F`，繼承要先關掉：

```powershell
icacls C:\ProgramData\ssh\administrators_authorized_keys /inheritance:r
icacls C:\ProgramData\ssh\administrators_authorized_keys /grant SYSTEM:F
icacls C:\ProgramData\ssh\administrators_authorized_keys /grant Administrators:F
```

**判斷順序**：先查 `net localgroup administrators` 有沒有這個帳號 →
有 → 用 `administrators_authorized_keys`；沒有 → 用一般的 `%USERPROFILE%\.ssh\authorized_keys`。
腳本已經自動做這個判斷。

## 多層 shell 引號地獄

從本機 PowerShell 經 `plink` 到遠端 `cmd.exe` 再到遠端 `powershell` 是三層 shell，任何巢狀雙引號
或反斜線路徑都容易在中間被吃掉或錯誤解析（例如 `Get-Acl 'C:\Users\James\.ssh'` 會變成單獨的 `\`
造成 `CommandNotFoundException`）。**一律用 Base64 EncodedCommand 繞過**，腳本裡的
`Invoke-RemotePS` 函式就是這個技巧：

```powershell
$bytes = [System.Text.Encoding]::Unicode.GetBytes($script)
$encoded = [Convert]::ToBase64String($bytes)
& $plink -ssh -batch -pw $Password $target "powershell -NoProfile -EncodedCommand $encoded"
```

## 陷阱：pubkey 明明登入成功，執行指令卻沒輸出、`Exit status -1`

**症狀**：`ssh -p 2222 user@host "echo OK"` 沒有任何輸出就結束，exit code 255；不帶指令的
互動式 `ssh user@host`（或用 heredoc 塞空白 stdin）直接卡死，超過 timeout 也不會自己斷開，必須手動
砍掉背景工作。用 `ssh -v` 看，其實 `debug1: Authenticated to ... using "publickey"` 早就成功了，
往下只看到：

```
debug1: Sending command: echo OK
debug1: channel 0: free: client-session, nchannels 1
debug1: Exit status -1
```

`Exit status -1` 代表 channel 在拿到指令的實際 exit code 之前就被關掉了 —— 這不是 pubkey 認證失敗
（認證那段早就過了），純粹是 Windows OpenSSH server 幫這個連線配置 pty/console 時的問題，跟金鑰或
ACL 無關，不要往 `authorized_keys` 那邊回頭排查。

**排除方式**：

1. 不要用不帶指令的裸 `ssh user@host` 去測試免密碼登入是否成功 —— 那會嘗試開互動式 shell，
   在這類有問題的 console 配置下容易直接掛住不回應。一律帶一個實際指令：`ssh user@host "whoami"`。
2. 一定要包一層 timeout（Bash 工具本身無法可靠中斷卡住的 ssh），例如 `timeout 15 ssh ...`，避免
   工具真的卡死要手動 TaskStop。
3. 就算沒有 timeout 包裝、單看 `-v` log 顯示 `Exit status -1` 且沒有指令輸出，也不代表登入失敗 ——
   重跑一次同樣指令（尤其加上 timeout）常常就正常回傳，輸出裡甚至會帶一堆 VT100 逸出碼
   (`[?9001h[?1004h[?25l[2J...`)，這是 Windows console 開頭的初始化序列，指令結果照樣夾在裡面，
   exit code 也會是正確的 `0`。判斷連線是否真的失敗，看 exit code 和有沒有預期輸出，不要只看
   log 尾端那行 `Exit status -1`。

## 除錯用：讀取 OpenSSH 事件記錄

當 pubkey 登入被拒但看不出原因時，先確認遠端帳號有沒有讀取事件記錄的權限（有時普通帳號也能讀），
再看最近的記錄（含密碼登入/失敗、host key 變更等，但如前述，query 階段被拒的 pubkey 不會出現）：

```powershell
Get-WinEvent -LogName "OpenSSH/Operational" -MaxEvents 60 | Select-Object TimeCreated, Id, Message
```

輸出如果是亂碼（Big5/UTF-8 編碼問題），內容通常還是能看懂關鍵字（`Accepted password`、
`Failed password`、`Connection closed ... [preauth]`）。

## Quick Reference

| 情況 | 判斷方式 | authorized_keys 位置 |
|---|---|---|
| 一般使用者 | `net localgroup administrators` 沒有這個帳號 | `%USERPROFILE%\.ssh\authorized_keys` |
| Administrators 成員（含內建 Administrator） | 有列在 `net localgroup administrators` | `C:\ProgramData\ssh\administrators_authorized_keys`（嚴格 ACL） |

## Common Mistakes

- 用 sshpass 測密碼登入失敗就以為密碼錯了 / 帳號被鎖 —— 先換 plink 排除工具本身的問題。
- 帳號密碼連續試錯太多次（尤其對 `Administrator`）可能觸發 Windows Account Lockout，之後
  密碼登入也會一起失敗，此時只能用 RDP/主控台解鎖。
- 只檢查一般使用者的 `~/.ssh/authorized_keys`，沒發現目標帳號其實也在 Administrators 群組裡。
- 巢狀引號直接照抄本機指令丟給 plink，結果在 cmd/powershell 中間被吃掉一部分。

---

## Conformance Addendum

## When to Use
Use when setting up passwordless SSH (public key) login to a Windows OpenSSH server from this machine, or when public-key auth to a Windows box is silently rejected despite correct ACLs and file content — covers "Permission denied (publickey,password,keyboard-interactive)", sshpass failing on Windows, and the administrators_authorized_keys trap for accounts in the local Administrators group.

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

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
