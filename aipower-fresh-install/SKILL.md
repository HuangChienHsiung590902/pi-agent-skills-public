---
name: aipower-fresh-install
description: 用官方安裝包（D:\_暫時保留\ECP\ 底下的 aipower-windows-*/ecp-windows-* 發行包）在本機全新安裝一套獨立的 aipower/ECP 實例——互動式 CLI 安裝程式的 printf 餵答案手法、安裝完一定要 patch 的 mariaDB4j jar、已知可忽略的 TcContact SQL error、以及啟動前的 port 衝突檢查。用在：要在本機再裝一套新的 aipower/ECP 測試環境、或既有實例壞掉要用官方包重裝時。
---

# aipower/ECP 全新安裝（官方安裝包）

## 適用情境

本機曾用同一套流程裝出至少兩個獨立實例：`C:\Lab2\chainsea`（見 `ecp-lab2-instance` skill，2026-07 因孤兒表壞掉後用官方包重裝修復）與 `C:\Aipower`（本次新裝的測試/開發用實例）。**每次要在本機再裝一套全新、獨立資料庫的 aipower/ECP 時都走這個流程**，不要手動兜 Tomcat/DB。

## 安裝包在哪裡找

用 `es`（Everything MCP）比用 Bash `find` 快很多且不會因中文路徑/權限問題出錯：

```
mcp__es__es_search query="aipower ext:zip;rar;exe;7z" 或 foldersOnly 搜 "ecp-windows"/"aipower-windows"
```

目前已知的官方包（都在 `D:\_暫時保留\ECP\` 底下，注意這個資料夾名稱意味著隨時可能被清掉，找不到時要重新搜）：

- `D:\_暫時保留\ECP\OneDrive_2_2026-6-1\aipower-windows-7.3.12.5-20260430-164651\aipower-windows-7.3.12.5-20260430-164651\` — aipower 品牌，2026-04-30 打包，**目前的預設選擇**
- `D:\_暫時保留\ECP\AI3_Version\ecp-windows-8.5.03.02-20250409-105336\` — 較舊的 ecp 品牌版本，`C:\ECP`(AI3測試環境) 就是這包裝的
- 版本號不能直接比大小：ecp/aipower 是同一產品先後兩個品牌名稱，各自獨立編號序列。**判斷「新版」看打包時間戳，不是看版本號數字**。

包內根目錄都有 `install.bat`，本質是 `jre\bin\java -cp "lib/*" com.jeedsoft.quicksilver.toolset.pack.install.Install`——一支互動式 CLI 安裝精靈。

## 安裝前：檢查 port 是否會撞

本機已知會長期佔用的 port：

| 實例 | 路徑 | Port |
|---|---|---|
| Lab2（正式，對外 Cloudflare Tunnel） | `C:\Lab2\chainsea` | 22821/22822 |
| AI3（VideoPage/LiveKit 開發沙盒） | `C:\ECP` | 12821/12822 |
| 本次新裝 | `C:\Aipower` | 22821/22822（跟 Lab2 同號，兩者不能同時開） |

安裝包（如 aipower-windows-7.3.12.5）的**預設 port 寫死在 `config\package.xml` 的 `<httpPort>`/`<httpsPort>`**，裝之前一定要讀這個檔案確認預設值，跟使用者核對是否接受撞號（撞號代表兩套不能同時啟動，只能擇一運行）。

啟動前檢查 port 有沒有人在監聽：

```powershell
try { Get-NetTCPConnection -LocalPort 22821,22822 -ErrorAction Stop | Select LocalPort,State,OwningProcess } catch { "no listener" }
```

**注意**：`-ErrorAction SilentlyContinue` 在這個 harness 底下查無連線時仍會讓工具回報 exit 1（非真的失敗），要用 `try { ... -ErrorAction Stop } catch {}` 包起來才不會誤判成指令失敗。

## 執行安裝：printf 餵答案

互動式安裝精靈的問答順序固定是 7 步（第 1 步填路徑、第 8 步填 y，中間 6 步用 Enter 吃預設值即可，除非要改 port/模組）：

1. Installation Path → 目標路徑
2. Multi-tenant → 預設 n
3. Server Type → 預設 1（Tomcat）
4. HTTP Port → 預設值（讀 package.xml）
5. HTTPS Port → 預設值
6. Modules → 預設 A（全部模組）
7. DataSource Configuration → 預設 `config/datasource.xml`
8. 最終確認 → **必須明確輸入 `y`**（空白不算數，會卡住重問）

```bash
cd "<安裝包解壓後的根目錄>"
printf '<目標路徑>\n\n\n\n\n\n\ny\n' | "jre/bin/java" -cp "lib/*" -Ddebug=false com.jeedsoft.quicksilver.toolset.pack.install.Install > "<可寫入的log路徑>" 2>&1
```

**踩坑：log 檔不能直接寫到 `C:\` 磁碟根目錄**（例如 `C:/install.log`），這個 bash 環境對磁碟根目錄沒有寫入權限，會得到 `Permission denied` 且整條指令直接失敗、看不到任何安裝輸出。log 要導到使用者的 scratchpad 目錄或其他已知可寫路徑。

安裝過程會跑很久（要灌大量 SQL），用 `run_in_background: true` 背景執行，等通知完成再讀 log。

## 已知可忽略的錯誤

安裝結束通常會印出：

```
Install finished with 4 SQL error(s), please view the log.
```

根因是 `TcContact`（CRM 聯絡人表）欄位太多，單行超過 MariaDB InnoDB 8126 bytes 的 row size 上限，導致該表 `create table`/後續 `create index`/`alter table` 連鎖失敗。這是安裝包本身既有的 bug，跟服務諮詢/文字客服/DeepSeek 等功能無關，**可以放著不用管**。

安裝完成會印出管理員密碼，格式：

```
Administrator's password is <password>, please modify it as soon as possible.
```

**務必截取這行回報給使用者**，並提醒盡快改密碼（見 `ecp-pwd` skill）。

## 安裝後必做：patch mariaDB4j jar

跟任何新裝的 aipower 一樣，**不 patch 的話每次重啟都會卡在「Data directory is not empty」死循環**。用 `ecp-server-startup` skill 附的 `patch_install.py`，對兩個相同檔名的 jar 都要 patch（一個在 webapp 底下、一個在 tool 底下，兩個都會被用到）：

```powershell
$py = "C:\Users\HCH\.claude\skills\ecp-server-startup\patch_install.py"
python $py "<安裝路徑>\apache-tomcat\webapps\aipower\WEB-INF\lib\mariaDB4j-core-3.1.0.jar"
python $py "<安裝路徑>\tool\lib\mariaDB4j-core-3.1.0.jar"
```

## 驗證安裝成功

1. 啟動 `<安裝路徑>\server.bat`（背景視窗）
2. 用 polling loop（不要用一串 `sleep`）等 HTTP port 進入 LISTEN 狀態，設合理逾時（如 90 秒）：
   ```powershell
   $deadline = (Get-Date).AddSeconds(90)
   while ((Get-Date) -lt $deadline) {
     try { if (Get-NetTCPConnection -LocalPort <port> -State Listen -ErrorAction Stop) { break } } catch {}
     Start-Sleep -Seconds 5
   }
   ```
3. `curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:<port>/aipower/` 預期回 `200`

三者都過，才算安裝與啟動都驗證成功。

## 已知坑：Tomcat 意外中斷後的殘留行程會擋下次啟動

`server.bat` 預設只是 `apache-tomcat/bin/catalina run`（前景執行），本身不管理子行程。如果 Tomcat 因故（例如視窗被強制關掉、當機）非正常結束，**它啟動的內嵌 MariaDB4j 子行程 `mariadbd.exe` 常常沒有跟著死掉**，會繼續佔著 `<安裝路徑>\mariadb\data` 的檔案鎖。下次執行 `server.bat` 時，新的 Tomcat 會嘗試在同一個 datadir 上再啟動一個 `mariadbd.exe`，因為資料目錄被舊行程鎖住而啟動失敗，整套服務起不來——而且 `netstat`/`tasklist` 表面上看起來一切正常（HTTP port 沒人聽、也沒有明顯錯誤訊息），容易誤以為是別的問題。

**修法：在 `server.bat` 執行 `catalina run` 之前，先偵測並 kill 掉屬於「這個安裝路徑」的殘留 `java.exe`/`mariadbd.exe`。** 關鍵是要能精準判斷某個行程是否屬於這套安裝——因為同一台機器上可能同時跑著 Lab2、AI3 等其他 aipower/ECP 實例，它們的 `java.exe`/`mariadbd.exe` 行程名稱完全一樣，**絕對不能用行程名稱做無差別 kill**，否則會誤殺其他實例。

**踩坑（v1 bug，已修正）：不要用 `CommandLine` 判斷 `java.exe` 是否屬於這套安裝。** `server.bat` 是用**相對路徑**設定 `CATALINA_HOME=apache-tomcat`（而不是絕對路徑），所以 Tomcat 啟動時 `java.exe` 的 `CommandLine` 裡只會出現 `apache-tomcat` 這種相對字串（例如 `-Dcatalina.base="apache-tomcat"`），**永遠不會包含安裝路徑本身**。第一版腳本用 `CommandLine -like "*<安裝路徑>*"` 判斷，結果只抓得到 `mariadbd.exe`（它是 MariaDB4j 用 Java 程式碼組出絕對路徑，所以指令列裡看得到），卻完全抓不到殘留的 Tomcat `java.exe`——實測時因此讓一個舊 Tomcat 繼續佔著 port，新的一輪 `server.bat` 啟動直接 `BindException: Address already in use` 失敗。

**正確做法：改用 `ExecutablePath`。** Windows 會把它解析成程序實際執行檔的**絕對路徑**（例如 `C:\Aipower\jre\bin\java.exe`），不管啟動時命令列寫的是相對還是絕對路徑都一樣準——因為每套安裝都有自己獨立的一份 `jre`，`mariadbd.exe` 也是解壓到各自安裝路徑底下的 `apache-tomcat\temp\MariaDB4j\base\bin\` 裡，兩者的 `ExecutablePath` 天生就跟安裝路徑綁定。

在安裝路徑根目錄新增 `kill-leftover.ps1`：

```powershell
$installDir = Split-Path -Parent $MyInvocation.MyCommand.Path
$installDir = $installDir.TrimEnd('\')

$procs = Get-CimInstance Win32_Process | Where-Object {
    ($_.Name -eq 'java.exe' -or $_.Name -eq 'mariadbd.exe') -and
    $_.ExecutablePath -like "$installDir\*"
}

if ($procs) {
    foreach ($p in $procs) {
        Write-Host "Killing leftover $($p.Name) (PID $($p.ProcessId))"
        Stop-Process -Id $p.ProcessId -Force -ErrorAction SilentlyContinue
    }
    Start-Sleep -Seconds 2
} else {
    Write-Host "No leftover processes found."
}
```

`server.bat` 開頭（在設定 `CATALINA_HOME` 之前）加一行呼叫它：

```bat
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0kill-leftover.ps1"
```

用 `Split-Path -Parent $MyInvocation.MyCommand.Path` 取腳本自身所在目錄，而不是寫死路徑，這樣同一份 `kill-leftover.ps1` 複製到不同安裝路徑（例如以後又裝一套 `C:\Aipower2`）也能直接用，天然只匹配自己這套的行程。

**實測驗證過兩輪**：
- 第一輪（驗證 `mariadbd.exe` 清理）：在 `C:\Aipower` 上先讓 Tomcat 意外中斷、確認殘留 `mariadbd.exe` 還活著，套用 v1 腳本後重新執行 `server.bat`，該殘留行程被自動偵測並強制結束，Tomcat 正常啟動、port 進入 LISTEN、`curl` 回應 200。
- 第二輪（意外抓到 v1 的 `java.exe` 判斷 bug）：修改 `application.properties` 後要重啟套用設定，結果 v1 腳本沒抓到殘留的 Tomcat `java.exe`（只殺了 `mariadbd.exe`），導致新的 `server.bat` 因 port 被舊 Tomcat 占著而 `BindException` 失敗、`curl` 回應 500。手動用 `Get-CimInstance Win32_Process` 確認兩個 `java.exe`（一個舊的、一個因 bind 失敗但沒真正結束的新的）都還活著、`ExecutablePath` 都指向 `C:\Aipower\jre\bin\java.exe`，改用 `ExecutablePath` 判斷的 v2 腳本後重新測試，`kill-leftover.ps1` 正確抓到並清掉兩個 `java.exe` + 殘留 `mariadbd.exe`，乾淨重啟後 port 正常 LISTEN、`curl` 回應 200，且 `catalina.log` 不再出現 `BindException`。

## 相關 skill

`ecp-lab2-instance`（這套流程第一次被完整記錄的來源，含孤兒表壞掉的教訓）、`ecp-ai3-instance`（另一套用不同版本安裝包裝出來的實例）、`ecp-server-startup`（`patch_install.py` 來源、mariaDB4j 相關陷阱總覽）、`ecp-pwd`（重設/變更管理員密碼）。

## 待確認事項

- `D:\_暫時保留\` 這個資料夾未來若被清掉，上面列的安裝包路徑就會失效，屆時要重新用 `es` 搜尋。
- 目前只驗證過 `aipower-windows-7.3.12.5` 這包的完整流程；`ecp-windows-8.5.03.02` 版本號雖然抓到 `install.bat`，但沒有實際跑過這份流程，模組清單/問答步驟數可能不完全一樣，遇到時要留意差異。

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
