---
name: portable-sshd-frp
description: Build/run portable-sshd (D:\CODES\portable-sshd) and its pro build with built-in frp reverse proxy — expose local SSH/SFTP through an frps server
triggers:
  - portable-sshd
  - sshd-portable-pro
  - frp
  - frps
  - frpc
argument-hint: "[build|run|connect]"
---

# portable-sshd-frp Skill

## Purpose

`D:\CODES\portable-sshd` is a single-exe, install-free Windows SSH server
(no service, no admin rights, no OpenSSH Windows feature). It has two build
variants selected by a Go build tag:

| Build | Command | Size | frp |
|---|---|---|---|
| Standard | `go build -o sshd-portable.exe .` | ~7.3 MB | not included |
| Pro | `go build -tags pro -o sshd-portable-pro.exe .` | ~23 MB | `github.com/fatedier/frp` client built in |

The pro build can connect out to an **frps** (frp server) and expose this
machine's SSH port under a remote port on the frps side — this is how you'd
reach this machine's SSH from outside without port-forwarding/opening
firewall holes on this box directly.

## Key Files

| File | Purpose |
|---|---|
| `D:\CODES\portable-sshd\main.go` | SSH server (conpty shell + `pkg/sftp` subsystem), parses `-frp-*` flags |
| `D:\CODES\portable-sshd\frp_pro.go` | `//go:build pro` — real `startFRP()` using `fatedier/frp/client` |
| `D:\CODES\portable-sshd\frp_stub.go` | `//go:build !pro` — stub `startFRP()` that `log.Fatal`s telling you to use the pro build |
| `D:\CODES\portable-sshd\authorized_keys` | pubkey allowlist (also supports Windows password login via `LogonUser`) |
| `D:\CODES\portable-sshd\host_key` | auto-generated ed25519 host key |

## Pro-build frp flags

```
-frp-server <host:port>    frps address (leave empty = frp disabled, behaves like standard build)
-frp-token <token>         frps auth.token
-frp-remote-port <port>    port frps opens publicly for this tunnel (default: same as -addr's local port)
```

Example:
```
sshd-portable-pro.exe -addr :2222 -frp-server frps.example.com:7000 -frp-token xxxx -frp-remote-port 6022
```

Connecting from outside then targets **the frps machine's address**, not
this machine's local IP:
```
ssh  -p 6022 <winuser>@frps.example.com
sftp -P 6022 <winuser>@frps.example.com
```

## Build & smoke test (verified working 2026-08-01)

```bash
cd D:/CODES/portable-sshd
go build -o sshd-portable.exe .            # standard
go build -tags pro -o sshd-portable-pro.exe .   # pro
```

End-to-end tested with a self-built `frps.exe` (frp v0.70.1, cloned +
`go build ./cmd/frps`, had to stub an empty `web/frps/dist/` dir for the
`//go:embed dist` directive to compile) — `ssh`/`sftp` through the tunnel's
remote port both worked, file content round-tripped correctly via
`put`/`get`.

## 已修復陷阱：`exec` request 沒過 shell 解析，`echo`/`dir` 等內建指令必失敗（2026-08-02 修復）

**症狀**：`ssh -p 2222 user@host "echo hello"`、`"dir"`、`"whoami /priv"` 這類帶 shell 內建指令
或參數的指令完全沒有輸出、exit code 255，`ssh -v` 顯示 pubkey 認證早就成功
（`Authenticated to ... using "publickey"`），之後只看到 `Exit status -1` 就斷線。但
`hostname`、`whoami`（不帶參數）、`cmd /c echo hello` 這種本身就是獨立 .exe 或已經包一層
shell 的指令卻能正常執行。

**別被誤導去查 Windows 內建 OpenSSH**：這個現象很像 Win32-OpenSSH 的
`HKLM\SOFTWARE\OpenSSH\DefaultShell` / `DefaultShellCommandOption` 沒設好，但如果目標埠是
`portable-sshd`/`sshd-portable-pro.exe` 在監聽（不是標準 22 埠、`Get-Process -Id
<PID監聽該埠>` 查出來 Path 是 `D:\sshd-portable-pro.exe` 而非
`C:\Windows\System32\OpenSSH\sshd.exe`），改 Windows 登錄檔完全無效——這個 server 是
`D:\CODES\portable-sshd\main.go` 自己 handle SSH protocol，不會去讀那組登錄檔。**先用
`Get-NetTCPConnection -LocalPort <port> | Get-Process` 確認是哪個 sshd 實作在監聽，再決定要
修哪邊。**

**根因**：`main.go` 的 `handleSession()` 在 `case "shell", "exec":` 分支裡，`exec` request
直接把整段指令字串（例如 `echo hello`）丟給 `conpty.Start(cmdLine, ...)`，等同直接呼叫
Windows `CreateProcess`——完全沒有經過任何 shell 解析。`echo`、`dir`、`cd`、管線符號這些 shell
內建語法找不到對應的獨立 `.exe`，spawn 直接失敗，程式只把錯誤寫進自己的 log
（`log.Printf("spawn %q: %v", ...)`）就關閉 channel，從未送出 `exit-status`，這就是客戶端看到
`Exit status -1` 且完全沒有輸出的原因。

**修法**（已套用在 `D:\CODES\portable-sshd\main.go`）：新增 `wrapInShell(shell, cmd string)
string` helper，把 `exec` request 的原始指令字串包進 `-shell` 參數指定的 shell 再丟給
`conpty.Start()`：

```go
case "shell", "exec":
    cmdLine := shell
    if req.Type == "exec" {
        cmdLine = wrapInShell(shell, parseExecCommand(req.Payload))
    }
```

`wrapInShell` 依 shell 檔名判斷包法（`powershell`/`pwsh` → `-NoLogo -NoProfile -Command
"<cmd>"`；`cmd.exe` → `/c "<cmd>"`；其他預設 `-c "<cmd>"`），`quoteArg` 負責把內嵌雙引號
escape 掉，避免指令字串裡有引號時把整個命令列拆斷。單元測試在
`D:\CODES\portable-sshd\shellwrap_test.go`（`go test -run TestWrapInShell -v .`）。

**部署注意**：改完 `main.go` 要 `go build -tags pro -o sshd-portable-pro.exe .` 重新編譯，
執行中的 exe 會被 Windows 鎖檔案，複製新版前要先在遠端把跑著的 `sshd-portable-pro.exe`
行程結束掉，複製完再重新啟動，2222 埠的連線才會套用新版行為。

**驗證修好與否的判斷方式**：`ssh -p 2222 user@host "echo hello; dir; whoami"` 有正常輸出且
`echo $?`／exit code 是 0（不是 255）。若改用 PowerShell 7+ 語法的 `&&` 分隔符號會在 Windows
PowerShell 5.1（`-shell` 預設值）上報 `The token '&&' is not a valid statement separator`，
這其實代表指令**已經**正確送進 PowerShell 解析執行了（是真正的語法錯誤，不是連線問題）——用
`;` 分隔多指令即可。

## 密碼登入身份切換：只有非 pty `exec` 才會用連線帳號執行（2026-08-02 新增）

**使用者需求**：「我希望它就跟一般的 SSHD 一樣」——ssh 連進來執行指令時，應該用登入帳號自己的
Windows 身份，而不是永遠用跑 `sshd-portable-pro.exe` 這個行程本身的帳號權限。

**採用範圍（使用者明確選定，非全功能對等）**：

| 登入方式 | Session 類型 | 執行身份 |
|---|---|---|
| 密碼登入 | `exec`（非互動，例如 `ssh host "cmd"`） | **切換成該登入帳號**（`CreateProcessWithLogonW`） |
| 密碼登入 | `shell`（互動 pty） | 仍是服務行程自己的帳號（不變） |
| 公鑰登入 | `exec` 或 `shell` | 仍是服務行程自己的帳號（不變） |

**為什麼不是全部都切換**：真正的 sshd（Win32-OpenSSH）用 S4U logon 幫所有登入方式做身份切換，
但那需要 `SeTcbPrivilege`，實務上等於要求服務以 SYSTEM 執行——跟這個專案「不要 service、不要
admin 權限」的設計目標直接衝突。折衷方案是用 `CreateProcessWithLogonW`（就是 Windows `runas`
背後那個 API）：**不需要呼叫端有管理員權限**就能用另一個帳號的帳密啟動行程，代價是它只吃
`STARTUPINFOW`，不支援 `STARTUPINFOEXW` 的擴充屬性列表，所以**沒辦法掛 ConPTY 偽終端機**——這就是
為什麼互動式 `shell` 只能維持原樣，只有非 pty 的 `exec` 能吃這個身份切換。

**實作位置**（`D:\CODES\portable-sshd\main.go`）：

1. `PasswordCallback` 把明文密碼透過 `ssh.Permissions.Extensions["auth-password"]` 帶到後面的
   session handler（公鑰登入這個 callback 不會設，所以 `winPassword == ""` 就是公鑰登入的判斷依據）。
2. `handleConn`/`handleSession` 多帶 `winUser`, `winPassword` 兩個參數。
3. `"exec"` 分支：`winPassword != ""` 時呼叫新的 `runImpersonatedExec()`，否則走原本
   `conpty.Start()` 路徑；`"shell"` 分支完全獨立出來，永遠走 `conpty.Start()`，不受密碼登入影響。
4. `runImpersonatedExec()` 用手動宣告的 `procCreateProcessWithLogonW`
   （`advapi32.dll!CreateProcessWithLogonW`，`x/sys/windows` 沒有現成綁定，比照現有
   `procLogonUserW` 的宣告方式），I/O 用一般匿名 pipe（`windows.CreatePipe` + 手動
   `SetHandleInformation` 控制繼承)，接到 `channel`/`channel.Stderr()`，跑完用
   `WaitForSingleObject` + `GetExitCodeProcess` 拿 exit code 回傳。

**自我檢查測試**：`impersonate_test.go` 用一個不存在的帳密呼叫 `runImpersonatedExec`，確認整條
API 呼叫/pipe 建立/清理路徑在失敗時乾淨回傳 error（不會 panic 或卡死），不需要真的建一個測試帳號。

**驗證方式**（部署後，由使用者手動換掉遠端的 exe）：用一個**非管理員、非 hch** 的本機 Windows
帳號密碼登入，跑 `ssh -p 2222 <其他帳號>@host "whoami"`，應該回傳那個帳號，而不是
`lenovo\hch`（服務行程本身的帳號）。互動式 `ssh -p 2222 <其他帳號>@host` 進 shell 則仍然是
`hch` 身份——這是設計範圍內的限制，不是 bug。

## 已排除的假陽性：驗證身份切換時，本機金鑰會搶答成 pubkey 登入（2026-08-02）

**症狀**：明明已經照上面部署了新 exe，跑 `ssh -p 2222 james@10.145.119.8 "whoami"` 卻還是回傳
`lenovo\hch`，看起來身份切換完全沒生效。

**根因（不是身份切換的 bug）**：OpenSSH client 預設會**依序自動嘗試** `~/.ssh/id_rsa`、
`id_ecdsa`、`id_ecdsa_sk`、`id_ed25519` 這些預設金鑰檔——不管你在指令列打的使用者名稱是誰。只要
本機 `~/.ssh/id_ed25519.pub` 剛好在伺服器的 `authorized_keys` 允許清單裡，client 就會自動用
**pubkey** 認證成功，完全不會走到密碼提示這一步。而 `main.go` 的 `PublicKeyCallback` 本來就沒有
檢查連線時打的帳號名稱要不要跟金鑰對應——這是本專案既有設計，不是這次身份切換功能引入的新問題。
所以這次連線其實走的是公鑰登入，依照設計就是用服務行程自己的帳號（`hch`）執行，行為完全正確。

**判斷方式**：先加 `-v` 看認證方式：

```
ssh -v -p 2222 james@10.145.119.8 "whoami"
```

log 尾端如果出現：

```
debug1: Offering public key: /c/Users/HCH/.ssh/id_ed25519 ...
debug1: Server accepts key: ...
Authenticated to ... using "publickey".
```

代表這次是 pubkey 登入，回傳服務帳號是預期行為，不是 bug。

**正確的驗證指令**：要真正測到密碼登入的身份切換，必須加
`-o PubkeyAuthentication=no` 擋掉自動送金鑰，強制走密碼提示：

```
ssh -p 2222 -o PubkeyAuthentication=no james@10.145.119.8 "whoami"
```

**這條指令不能用 Bash/PowerShell 工具直接跑**——工具沒有真正的 TTY，OpenSSH 的密碼提示需要讀
`/dev/tty`，會直接報 `read_passphrase: can't open /dev/tty: No such device or address`，
密碼還沒機會輸入就被跳過、連續 3 次後顯示 `Permission denied (password,publickey)`，看起來像
認證失敗，但實際上根本沒送出密碼。**改用 `plink`**（PuTTY 附的，通常在
`C:\Program Files (x86)\PuTTY\plink.exe`）就能在 Bash/PowerShell 工具裡直接測，不需要請使用者
手動操作：

```bash
"/c/Program Files (x86)/PuTTY/plink" -ssh -batch -P 2222 -pw <密碼> <user>@<host> "whoami"
```

第一次連線若跳 host key 確認會卡住，先 `printf "y\n" | plink ...`（不加 `-batch`）跑一次存進快取。
（這跟 `ssh-passwordless-windows` 技能裡處理密碼登入 plink 陷阱的手法一致。）

## 已驗證：部署後密碼身份切換 end-to-end 測試（2026-08-02）

用上面 `plink` 的方法實際部署後測試，結果：

| 測試 | 帳號 | 指令 | 結果 |
|---|---|---|---|
| 公鑰登入回歸測試 | `hch`（pubkey） | `whoami; echo hello; dir` | `lenovo\hch`，內建指令正常，exit 0 |
| 密碼登入身份切換 | `james`（password） | `whoami` | `lenovo\james`（非 `hch`，切換成功），連測兩次穩定 |
| 密碼登入 + shell 內建指令 | `james`（password） | `echo hello; whoami; dir` | 輸出正常，身份仍是 `james` |
| exit code 傳遞 | `james`（password） | `exit 3` | 客戶端正確收到 exit code 3 |
| 密碼登入互動式 pty shell | `james`（password） | `whoami`（互動式） | `lenovo\hch`（維持服務帳號，符合設計範圍，不是 bug） |

功能運作符合預期。

## 陷阱：即使帳號在 Administrators 群組，寫 HKLM 還是會被拒絕（UAC Admin Approval Mode）

**症狀**：用密碼身份切換 exec，連線帳號（例如 `hch`）明明是本機 Administrators 群組成員，
`whoami` 也確實回傳該帳號，但執行 `reg add HKLM\...` 卻回傳 `ERROR: Access is denied.`（exit 1）。
讀取（`reg query`）完全正常，只有寫入被拒。

**根因**：這不是 `CreateProcessWithLogonW` 或這個 SSH server 的 bug，是 Windows UAC 的
**Admin Approval Mode** 設計本身：只要帳號是「Administrators 群組成員，但不是內建 Administrator
（SID 結尾 `-500`）」，透過互動式登入取得的權杖預設就會被過濾成標準使用者權限，程式必須經過
明確的「以系統管理員身分執行」（UAC 同意提示）才能拿到完整權杖去寫 `HKLM`。這一層過濾發生在
權杖建立當下，`CreateProcessWithLogonW` 拿到的也是被過濾後的權杖，跟呼叫方式無關。

**已驗證的解法**：改用**內建 Administrator 帳號**（`SID -500`，`net user administrator` 可查
`Account active: Yes`/`No`）登入。這個特殊帳號預設不受 Admin Approval Mode 過濾，用它做密碼身份
切換 exec，`reg add` 寫入 `HKLM` 直接成功：

```bash
"/c/Program Files (x86)/PuTTY/plink" -ssh -batch -P 2222 -pw <Administrator密碼> administrator@<host> \
  "reg add HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR /v Start /t REG_DWORD /d 4 /f"
```

（上例把 `USBSTOR` 的 `Start` 值改成 `4` 停用 USB 儲存裝置讀取，改回 `3` 即恢復。這組登錄檔鍵值
本身跟這個 skill 無關，只是拿來驗證寫入 HKLM 有沒有成功的實例。）

**如果要讓 `hch`（而非內建 Administrator）也能寫 HKLM**，選項（皆需先用 RDP 主控台 session 手動
設定一次，SSH 密碼身份切換無法繞過這第一步）：

1. **全機關閉 UAC**（`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` 的
   `EnableLUA` 改 `0`，需重開機）——`hch` 之後永遠拿完整權杖，但是**全機、永久性**的安全性降級，
   所有程式都失去 UAC 保護，不建議。
2. **預先建立一個「以最高權限執行」的 Scheduled Task**（勾 "Run with highest privileges"，存
   `hch` 帳密），之後透過 SSH 用 `schtasks /run /tn <工作名稱>` 觸發即可繞過過濾——影響範圍只限
   這個工作本身，比整機關 UAC 安全，但仍是預先開的特權通道，需自行評估風險。
3. **維持現狀，改用內建 Administrator 帳號登入**（已驗證可行，見上）——不需改任何系統設定，
   代價是要另外管理一組特權帳密，不是 `hch` 自己的密碼。

三個選項都涉及該台機器整體安全性設計的取捨，不會主動去改，需使用者自行決定要不要做、選哪個。

## 應用範例：遠端開關 USB 儲存裝置讀取（已驗證 2026-08-02）

靠密碼身份切換 + 內建 Administrator 帳號（見上一節），可以直接用 SSH 遠端開關這台機器的 USB
隨身碟/外接硬碟讀取功能，不需要 RDP。改的是 `USBSTOR` 服務的 `Start` 值：

| Start 值 | 效果 |
|---|---|
| `3` | 開啟（正常讀取 USB 儲存裝置） |
| `4` | 關閉（USB 儲存裝置無法被系統辨識/讀取） |

```bash
PLINK="/c/Program Files (x86)/PuTTY/plink"

# 關閉
"$PLINK" -ssh -batch -P 2222 -pw <Administrator密碼> administrator@<host> \
  "reg add HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR /v Start /t REG_DWORD /d 4 /f"

# 開啟
"$PLINK" -ssh -batch -P 2222 -pw <Administrator密碼> administrator@<host> \
  "reg add HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR /v Start /t REG_DWORD /d 3 /f"

# 查詢目前狀態
"$PLINK" -ssh -batch -P 2222 -pw <Administrator密碼> administrator@<host> \
  "reg query HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR /v Start"
```

**注意**：改設定不需要重開機，但已經插著的 USB 裝置通常要**拔掉重插**系統才會套用新狀態
（關閉後系統不會主動退掉正在使用的裝置，開啟後也不會主動重新辨識已插著但先前被拒絕的裝置）。

## TODO — not yet resolved

**How to find/reach the pro build's connect IP for the *current* real-world
setup is still open.** Depends on where frps actually lives:

- If frps runs on a VPS/cloud box with a public IP → that IP (or its
  hostname) is what external clients dial, on `-frp-remote-port`.
- If no frps is deployed yet → one needs to be stood up somewhere with a
  public IP/open port 7000 (or whatever `bindPort` is configured) before
  `-frp-server` on the pro build has anywhere real to connect to.
- Alternative if there's no VPS available: skip frp and use
  `cloudflare-tunnel` skill (`C:\Users\HCH\.claude\skills\cloudflare-tunnel\`)
  instead — same "expose local service to internet" goal via Cloudflare's
  edge, TCP tunneling works with `cloudflared access tcp` / private network
  routing, no VPS needed.

Next session: ask user where frps should run (existing VPS? new one?), then
fill in the actual `-frp-server` host + real `frps.toml` (`bindPort`,
`auth.token`) here.

---

## Conformance Addendum

## When to Use
Build/run portable-sshd (D:\CODES\portable-sshd) and its pro build with built-in frp reverse proxy — expose local SSH/SFTP through an frps server

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
