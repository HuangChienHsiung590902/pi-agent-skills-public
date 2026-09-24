---
name: sfaa-room-monitor-remote-control
description: 透過 SSH（port 2222，金鑰驗證）遠端連進跑 room_monitor.py 的 SFAA 機器（Gram），用 room_monitor_ctl.py 查詢/操作正在跑的 room_monitor 狀態（STATUS/RESTART/LOGOUT/READY/UNREADY），或執行任意 Windows 指令。當使用者要求「遠端查 room_monitor 狀態」、「SSH 進 SFAA 機器」、「遠端重啟 room_monitor」、「遠端切就緒/未就緒」、「room_monitor_ctl」、或提到 SFAA 機器連不上/要遠端操作時使用。也用於排查「Failed to start embedded python interpreter」這個 PyInstaller onefile bootloader 崩潰，以及重新打包部署 room_monitor.exe 時要不要清 _MEI 暫存目錄。
---

# SFAA room_monitor 遠端控制（SSH + room_monitor_ctl.py）

2026-07-24 建立。用途：讓你能從區網/ZeroTier 上的其他裝置，遠端連進跑 `room_monitor.py` 的機器（機器名 `Gram`），查詢它目前的監控狀態，或觸發重啟/登出/切就緒/未就緒。

**相關 skill**：`sfaa-ecp-room-monitor`（系統整體架構/已知坑）、`debug-vrs`（用原始碼跑 room_monitor.py + 接管 Chrome 除錯）。這份 skill 只講「遠端下命令」這一塊，跟現場操作的安全鐵則共用同一套（見下方「安全規則」）。

## 架構總覽

分兩層，各自負責不同事：

1. **Windows 內建 OpenSSH Server**（port 2222，只允許金鑰驗證，防火牆只開放區網 `192.168.100.0/24` 跟 ZeroTier `10.145.119.0/24`）——負責「任意指令」，這是真正的 SSH，沒有另外刻認證。
2. **`room_monitor.py` 內建的本機控制通道**（`control_server.py`，只綁 `127.0.0.1:8765`，完全不做自己的認證）——負責「查/控 room_monitor 目前狀態」，安全邊界完全依賴「只有本機能連到」這個前提（外部連線已經先被上面的 SSH 金鑰驗證擋過一層）。

```
遠端裝置 → ssh -p 2222 (金鑰驗證) → Windows OpenSSH Server → 拿到 Gram 的 shell
              → 任意 Windows 指令：直接在 shell 執行
              → room_monitor 專屬操作：跑 room_monitor_ctl.py
                    → 連 127.0.0.1:8765（room_monitor.py 內的 control_server.py）
                          → 執行對應動作 / 回傳目前狀態
```

## 怎麼連

```bash
ssh -p 2222 HCH@192.168.100.147     # 區網 Wi-Fi
ssh -p 2222 HCH@10.145.119.100      # ZeroTier overlay 網
```

金鑰驗證，不會跳密碼輸入（密碼登入已經在 `sshd_config` 關掉）。

**⚠️ 遠端 shell 預設是 cmd.exe，`cd D:\...` 不會真的切磁碟機**（cmd.exe 的行為，`cd` 沒加 `/d` 只換路徑不換磁碟機）。要嘛在指令裡加 `/d`，要嘛更簡單：**每次都直接用完整路徑**，不要依賴 `cd`：

```bash
ssh -p 2222 HCH@192.168.100.147 "C:\Users\HCH\AppData\Local\Programs\Python\Python312\python.exe D:\SFAA\room_monitor_ctl.py status"
```

單行指令可以直接夾在 `ssh ... "指令"` 裡（如上），不需要真的進去互動式 shell。

## room_monitor_ctl.py 指令

固定路徑：`D:\SFAA\room_monitor_ctl.py`（跟正式監控用的 `D:\SFAA\room_monitor.py` 放同一個目錄，因為要 import 同目錄的 `control_server.py`）。

```
python D:\SFAA\room_monitor_ctl.py status    # 查詢目前狀態（唯讀，隨時可測，零風險）
python D:\SFAA\room_monitor_ctl.py restart   # 觸發重新啟動（跟系統匣「重新啟動」一樣）
python D:\SFAA\room_monitor_ctl.py logout    # 觸發登出（跟系統匣「登出」一樣）
python D:\SFAA\room_monitor_ctl.py ready     # 手動切就緒
python D:\SFAA\room_monitor_ctl.py unready   # 手動切未就緒（建檔中）
```

回傳一段 JSON，`{"ok": true, ...}` 或 `{"ok": false, "error": "..."}`，exit code 跟著 `ok` 走（`ok:false` 時 exit 1，方便寫腳本判斷）。

**沒有 `login` 指令**——帳密登入本來就得手動在瀏覽器輸入（含 2FA/驗證碼），不會、也不該開放遠端做這件事。

### STATUS 回傳格式

```json
{"ok": true, "key": "ready", "rooms": 0, "code": "CS0006", "ready_elapsed": 42}
```

`key` 是四種狀態之一：`system_offline`(未登入系統) / `service_offline`(未登入服務) / `unready`(未就緒) / `ready`(就緒)。`ready_elapsed` 只有 `key=="ready"` 時才有值（秒數），其餘是 `null`。

## 安全規則（跟現場操作共用）

- **`READY`/`UNREADY` 只在你自己清楚知道現在的服務狀態、且有必要時才用**——這兩個指令效果等同直接在畫面上手動點按鈕，會影響真人是否會被派線。切到 `UNREADY` 永遠是安全方向（降低風險），切到 `READY` 則是讓帳號真的開始等派線，要謹慎。
- 跟現場操作同一條鐵則：**任何操作前，先用 `status` 確認目前真實狀態，別憑印象猜**。
- `LOGOUT`/`RESTART` 不會影響真人服務中的通話（`RESTART` 靠 guardian 在 ~10 秒內自動救回整套監控），但還是建議在確認沒有真人正在服務中的時段操作，尤其是 `RESTART`（會有幾秒鐘監控空窗）。

## 已知坑

### 1. `RESTART` 的回應時序（2026-07-24 修正過）

`restart_app()` 內部是 `os._exit(0)`，如果同步呼叫會讓 process 在把 `{"ok": true}` 寫回給客戶端之前就死掉，導致遠端每次「成功重啟」都被 `room_monitor_ctl.py` 誤判成連線失敗。現在的做法是 `_control_restart()` 用 `asyncio.get_running_loop().call_later(0.5, restart_app)` 延後 0.5 秒才真的退出，讓回應有時間寫完、flush、關閉連線。如果之後又改到 `restart_app`/`_control_restart` 這段，要記得保留這個「先回應、再退出」的順序。

### 2. `beforeunload` 對話框被錯誤處理（2026-07-24 修正過）

`room_monitor.py` 的全域 dialog handler 原本對所有對話框都用 `dismiss()`（=取消）。對一般 alert/confirm 沒差，但對 `beforeunload` 對話框，`dismiss()` 的語意是「留在原頁面」——會把 `_do_logout()` 自己想做的跳轉卡住，每次都 `net::ERR_ABORTED`。現在改成只有 `beforeunload` 用 `accept()`放行，其餘型別維持 `dismiss()`。這個 bug 是舊代碼本來就有的，不是這次新加的功能造成的，只是剛好在測 `LOGOUT` 指令時測出來。

### 3. PyInstaller onefile bootloader 崩潰：「Failed to start embedded python interpreter!」

這是 exe 打包成單一檔案（onefile）本身的通病，跟這次的遠端控制功能無關，但排查/重新部署時很容易撞到，記錄在這裡方便查：

**根因**：`taskkill /F` 強砍 `room_monitor.exe` 從不讓 bootloader 有機會清掉自己解壓到 `%TEMP%\_MEI######` 的暫存目錄。這些資料夾會越堆越多（實測過一次連續重啟後累積 15 個、共 1.4GB），拖慢下次啟動的解壓速度，也更容易被防毒軟體即時掃描鎖住其中幾個檔案——兩者疊加就是 guardian 自動重試 3 次都失敗、彈出這個錯誤視窗的主因。

**已經做的修正**：`_kill_stray_instances()`（guardian 重啟迴圈用）現在會在殺完殘留程序後，順便清空沒有任何 `room_monitor.exe` 還活著時的 `_MEI*` 暫存目錄，降低復發機率，不用每次都人工介入。

**手動排查步驟**（如果又發生）：
1. 關掉錯誤視窗，全部砍乾淨：
   ```powershell
   Get-CimInstance Win32_Process -Filter "Name='room_monitor.exe'" | ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
   Get-Process -Name chrome -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
   ```
2. 清空殘留的 `_MEI*` 暫存目錄（確認第 1 步已經砍乾淨、沒有任何 room_monitor.exe 還在跑之後）：
   ```powershell
   Remove-Item "$env:TEMP\_MEI*" -Recurse -Force -ErrorAction SilentlyContinue
   ```
3. 重新啟動：`Start-Process "D:\SFAA\fix\dist\room_monitor.exe"`

**根治方式（需要系統管理員權限，這個 session 通常沒有，要請使用者自己跑）**：把防毒即時掃描排除掉 dist 目錄跟 `_MEI*` 暫存目錄：
```powershell
Add-MpPreference -ExclusionPath "D:\SFAA\fix\dist"
Add-MpPreference -ExclusionPath "$env:TEMP\_MEI*"
Add-MpPreference -ExclusionProcess "room_monitor.exe"
```
若這台機器裝的是第三方防毒（非 Windows Defender），要去該防毒軟體自己的介面設定排除清單。

**真正徹底解決**這整類 onefile bootloader 問題的方法是 `sfaa-ecp-room-monitor` skill 裡提到的 Rust 改寫專案（已開發完成、待現場驗證，尚未切換正式環境）——原生單一 exe 不會有這個問題，但那是更大的工程，日常排查先用上面的手動步驟。

## 相關檔案位置

| 檔案 | 用途 |
|------|------|
| `D:\SFAA\control_server.py` / `D:\SFAA\room_monitor_ctl.py` | 正式監控用（跟 `D:\SFAA\room_monitor.py` 同目錄，`sfaa_watch.vbs` 啟動的 exe 其實是跑 `D:\SFAA\fix\dist\room_monitor.exe`，但這兩支 CLI 用的 control_server 是 `room_monitor.py`/`room_monitor.exe` 內建的，不管跑哪個都通） |
| `D:\SFAA\fix\control_server.py` / `D:\SFAA\fix\room_monitor_ctl.py` | PyInstaller 打包來源目錄的對應副本，改完 `D:\SFAA\fix\room_monitor.py` 要重新打包時，這兩個檔案要跟著同步（`room_monitor.py` 有 `from control_server import start_control_server`，同目錄才 import 得到） |
| `D:\SFAA\fix\` | **2026-07-24 起是 git repo**（`master` 分支），這次遠端控制功能的完整開發歷史都在裡面（`git log`），規格/計畫文件在 `docs/superpowers/specs/` 和 `docs/superpowers/plans/` |
| `C:\ProgramData\ssh\sshd_config` | SSH 設定（`Port 2222`、`PasswordAuthentication no`） |
| `C:\ProgramData\ssh\administrators_authorized_keys` | 允許登入的公鑰清單（HCH 是 Administrators 成員，Windows OpenSSH 規定要用這個檔案，不是一般的 `~/.ssh/authorized_keys`） |

## 驗證方式（改動後怎麼確認沒壞）

跑測試套件（`control_server.py`/`room_monitor_ctl.py` 有完整 pytest 覆蓋，`room_monitor.py` 本身沒有，要走 `debug-vrs` skill 手動驗證）：

```powershell
"C:\Users\HCH\AppData\Local\Programs\Python\Python312\python.exe" -m pytest "D:\SFAA\fix\tests\" -v
```

本機迴圈測試 SSH（不需要第二台裝置，驗證 sshd/金鑰設定本身）：
```powershell
ssh -p 2222 -o BatchMode=yes -o ConnectTimeout=5 HCH@127.0.0.1 whoami
```

完整端對端：照上面「怎麼連」那節，找一台區網內或 ZeroTier 上的裝置實測 `status`。

---

## Conformance Addendum

## When to Use
透過 SSH（port 2222，金鑰驗證）遠端連進跑 room_monitor.py 的 SFAA 機器（Gram），用 room_monitor_ctl.py 查詢/操作正在跑的 room_monitor 狀態（STATUS/RESTART/LOGOUT/READY/UNREADY），或執行任意 Windows 指令。當使用者要求「遠端查 room_monitor 狀態」、「SSH 進 SFAA 機器」、「遠端重啟 room_monitor」、「遠端切就緒/未就緒」、「room_monitor_ctl」、或提到 SFAA 機器連不上/要遠端操作時使用。也用於排查「Failed to start embedded python interpreter」這個 PyInstaller onefile bootloader 崩潰，以及重新打包部署 room_monitor.exe 時要不要清 _MEI 暫存目錄。

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
