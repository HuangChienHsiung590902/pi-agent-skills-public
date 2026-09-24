---
name: podman-windows
description: 在這台 Windows 機器上安裝/管理 Podman（WSL2 後端），以及讓 `--restart unless-stopped` 容器（例如 omniroute）在開機/登入後真的自動跑起來。當使用者提到「podman」「podman machine」「podman 開機啟動」「容器開機沒自動跑」時使用。
triggers:
  - podman
  - podman machine
  - podman 開機啟動
  - podman-restart.service
argument-hint: "[install|autostart|troubleshoot]"
---

# podman-windows Skill

## 前置需求：WSL2

Podman on Windows 靠 WSL2 跑 VM。全新機器通常還沒裝 WSL，`podman machine init` 會直接失敗（錯誤訊息可能因主控台編碼問題顯示成亂碼，實際內容是「找不到已安裝的發行版/需要執行 wsl --install」）。

判斷方式（避免主控台輸出亂碼誤判）：
```powershell
cmd /c "wsl --status > `"$env:TEMP\wsl_status.txt`" 2>&1"
Get-Content -Encoding Unicode "$env:TEMP\wsl_status.txt"
```

若沒裝，需要使用者自己在**系統管理員權限**視窗執行 `wsl --install`，然後**重新開機**（這是系統層變更，不要自動幫使用者做，請他自己執行）。

## 建立與啟動 machine

```powershell
podman machine init
podman machine start
```

`podman machine list` 可確認 VM 狀態。首次啟動後才能 `podman run`。

## 已知坑：`--restart unless-stopped` 容器不會真的在開機後自動跑

### 現象

容器建立時有帶 `--restart unless-stopped`，理論上應該要跟著 podman machine 一起自動起來，但實測發現：Windows 重開機後，即使有排程任務讓 `podman machine start` 成功執行（VM 確實起來了），容器仍然停在 `Exited` 狀態，需要手動 `podman start <name>`。

### 根因（已在此機器上實測確認，podman 6.0.2 / WSL）

VM 內負責這件事的 systemd unit `podman-restart.service`：

1. **預設是 disabled**，不會在 VM 開機時自動觸發。
2. 就算手動 `sudo systemctl enable podman-restart.service` 之後，它在真正冷啟動（`podman machine stop` → `podman machine start`）時仍會出現 race condition：`journalctl -u podman-restart.service` 可看到它跑到一半（做完 `system refresh` 之後、還沒真的呼叫 `podman start --all`之前）就收到 `Received shutdown.Stop(), terminating!` 被中止，導致容器沒有被啟動，但 service 本身仍回報 `Finished`/exit 0，看起來像成功了，容易誤判。

診斷指令：
```powershell
podman machine ssh "systemctl is-enabled podman-restart.service"
podman machine ssh "journalctl -u podman-restart.service --no-pager -n 30"
```

若看到 `system refresh` 後緊接著 `Received shutdown.Stop(), terminating!`，就是這個 race，不用再花時間查 VM 內部——直接用下面的外部補位方案。

### 解法：不依賴 VM 內部機制，改由 Windows 端明確補一道 start

`C:\Users\HCH\scripts\podman_autostart.py`（已存在，可直接沿用/複製到其他容器）：
- 先重試呼叫 `podman machine start` 直到成功或回報 already running
- 再對指定容器清單逐一重試 `podman start <name>`，同樣把 "already running" 視為成功
- 兩段都有重試（預設 12 次、間隔 5 秒），避免 API/machine 剛啟動還沒 ready 就打指令失敗

要追蹤/新增更多容器，編輯該檔案裡的 `CONTAINERS_TO_ENSURE_RUNNING` 清單即可。

### 排程任務設定

用「登入時觸發」（`-AtLogOn`），不是「開機時觸發」，因為 podman machine 的資料在使用者 profile 底下，系統帳號未必有權限/環境跑得起來：

```powershell
$pythonPath = (Get-Command python).Source
$action = New-ScheduledTaskAction -Execute $pythonPath -Argument '"C:\Users\HCH\scripts\podman_autostart.py"'
$trigger = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$env:USERNAME"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
Register-ScheduledTask -TaskName "Podman Machine AutoStart" -Action $action -Trigger $trigger -Settings $settings -Force
```

### 驗證方式（不要只看 LastTaskResult）

`podman-restart.service` 那次假成功就是因為只看了 exit code。正確驗證要真的模擬冷開機：

```powershell
podman machine stop
Start-Sleep -Seconds 5
Start-ScheduledTask -TaskName "Podman Machine AutoStart"
Start-Sleep -Seconds 40
podman ps -a    # 要看到目標容器是 Up，不是 Exited
```

## 常用指令備忘

```powershell
podman ps -a                        # 含已停止的容器
podman logs <name>
podman machine ssh "<linux 指令>"    # 直接在 VM 內執行指令查系統層問題
```

---

## Conformance Addendum

## When to Use
在這台 Windows 機器上安裝/管理 Podman（WSL2 後端），以及讓 `--restart unless-stopped` 容器（例如 omniroute）在開機/登入後真的自動跑起來。當使用者提到「podman」「podman machine」「podman 開機啟動」「容器開機沒自動跑」時使用。

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
