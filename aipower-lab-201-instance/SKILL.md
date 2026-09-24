---
name: aipower-lab-201-instance
description: >-
  Remote-manage the native (non-Docker) aipower/ECP install at 192.168.2.201
  (hostname WIN11-1, domain "lab", port 22821/22822) over SSH — the OpenSSH
  client on this operator machine mangles `domain\user` logins and must be
  replaced with PuTTY's plink/pscp; the embedded MariaDB4j port changes every
  restart and must be discovered from the running mariadbd.exe process; and
  Tomcat's server.bat uses relative paths so any detached/scheduled-task
  restart must `cd` into C:\aipower first. Also covers the full procedure for
  permanently removing a business feature/module (code + DB metadata + DB
  business data) from a live instance, and how to debug "can't connect"
  complaints without chasing phantom server-side bugs. Use when the user asks
  to SSH into ".201", work on C:\aipower on 192.168.2.201, or mentions this
  host by IP/hostname. NOT `ecp-lab2-instance` (different machine, C:\Lab2,
  fixed port 3306 DB) or `aipower-docker-local` (Docker containers).
---

# aipower @ 192.168.2.201 (native install, domain "lab")

> 這台就是 `cloudflare-tunnel` skill 裡記載的 **WIN11-1**（home-lab Windows 11
> client VM，`hch.james-huang.org` 對外服務的真正 origin，跑在這台的 22821）。
> 網域 `lab` 的網域控制站是 `SRV`（`192.168.2.200`）。對外路由/tunnel 設定去看
> `cloudflare-tunnel` skill，這份只管「怎麼 SSH 進去改東西」。

## 連線：一定要用 plink/pscp，不要用 OpenSSH client

這台機器的帳號是網域帳號（`lab\hch`、`lab\Administrator`），OpenSSH 的 `ssh`/`scp`
在把 `-l 'lab\hch'` 或 `lab\hch@host` 這種帳號傳給遠端 sshd 時，會把反斜線多跳脫一次
（remote 端實際收到 `lab\\hch`，變成不存在的帳號），導致密碼怎麼打都是
`Permission denied (publickey,password,keyboard-interactive)`，看起來很像密碼錯，
其實是 client 端的跳脫 bug。**改用 PuTTY 的 `plink`/`pscp` 就完全正常**，兩者都要
用 `-l` 分開帶帳號（不要塞進 `user@host` 那種寫法，一樣有同款跳脫問題）：

```bash
# 執行遠端指令
plink -ssh -pw '<password>' -l 'lab\hch' -hostkey "SHA256:DDMixmT5aNUTFtriubg2ccbsNxqslJUCouralrhQKMM" 192.168.2.201 "dir C:\aipower"

# 傳檔案（上傳）
pscp -pw '<password>' -hostkey "SHA256:DDMixmT5aNUTFtriubg2ccbsNxqslJUCouralrhQKMM" -l 'lab\hch' "本機路徑" "192.168.2.201:C:\aipower\目的檔名"
```

- `-hostkey` 帶固定值可以跳過互動確認 host key（batch 模式下沒帶會直接失敗:
  `FATAL ERROR: Cannot confirm a host key in batch mode`）。第一次連線要先跑一次
  `ssh -o StrictHostKeyChecking=accept-new` 或看 `plink` 報錯訊息拿到正確的
  SHA256 指紋，不要憑空套用上面這串（金鑰重灌後會變)。
- 密碼跟帳號要跟使用者要，不要寫死在這份技能檔裡。
- 多行/複雜 SQL 一律**寫成 .sql 檔案用 pscp 上傳再執行**，不要塞進單行
  `plink ... "cmd1 & cmd2 & cmd3"`——cmd.exe 對多行、含 `&&`/`^&^&` 跳脫、含 `~`
  開頭表名的指令常常解析出錯或整段被吃掉（`~lenustsfield` 這種表名尤其容易在
  bash 雙引號字串裡把反引號當成 command substitution 觸發，一定要寫檔上傳）。

## 環境配置

- 安裝路徑 `C:\aipower`，同時鏡射一份在 `D:\aipower`（同一份安裝，兩個磁碟代
  號指向一樣的內容，不是兩套環境）。
- `apache-tomcat`（http 22821 / https 22822）＋自帶 `jre` ＋自帶 `mariadb`
  （MariaDB4j 內嵌引擎），符合 `aipower-fresh-install` 官方安裝包的標準佈局。
- Web 應用只有編譯後的 `.class`（`WEB-INF\classes\com\...`），**沒有 .java 原始
  碼**——這是部署環境不是原始碼庫。要「改程式」實質上只能靠：(a) 反組譯/直接
  刪除 class 檔（適合純刪除功能），或 (b) 找到公司內部實際的原始碼庫另外改完
  重新編譯部署過來。開工前先跟使用者確認要哪一種。

## 資料庫：內嵌 MariaDB，port 每次重啟會變

連線設定寫在 `C:\aipower\apache-tomcat\extension\aipower\config\datasource.xml`
（`root` / 空密碼，`--skip-grant-tables` 啟動，理論上帳密檢查都不生效），但那個
`jdbc:marialocal:...` 是框架自己包的 driver，不能直接拿去給標準 mysql client 用。

**實際 TCP port 是隨機的，每次啟動都不同**，要從正在跑的 `mariadbd.exe` 行程參數現查：

```powershell
Get-CimInstance Win32_Process -Filter "Name='mariadbd.exe'" | Select-Object CommandLine
# 看 --port=xxxxx
```

mysql client 執行檔在 `C:\aipower\apache-tomcat\temp\MariaDB4j\base\bin\mysql.exe`
（同目錄還有 `mysqldump.exe`），不用另外裝：

```bash
plink ... 192.168.2.201 "C:\aipower\apache-tomcat\temp\MariaDB4j\base\bin\mysql.exe -uroot -P<port> -h127.0.0.1 default -e \"SELECT 1;\""
```

## ECP/aipower 標準 schema 速查（六大核心 + 選單樹）

- `TsUnit`：一個功能模組。`FCode`（如 `Ecp.UsbAgentMachine`）、`FName`、
  `FTable`（實際業務資料表，Tc/Tp/Tw/Tt 開頭，見下）、
  `FHomeClassName`/`FDaoClassName`/`FServiceClassName`/`FActionClassName`/`FApiClassName`
  = 六大核心類別全名，可以用 `FActionClassName LIKE '%關鍵字%'` 反查某個 Java package
  對應哪個 Unit。
- `TsPage`（`FUnitId`）→ `TsForm`/`TsFormField`/`TsFieldGroup`（`FFormId`）、
  `TsList`/`TsListField`/`TsUserListField`（`FListId`）、`TsField`（`FUnitId`）、
  `TsPrivilege`/`TsRoleUnitPrivilege`（`FUnitId`）、`TsToolItem`/`TsIconMenu`/
  `TsMobileMenu`/`TsScript`（`FPageId`）。
- `TsMenu`（`FPageId` 指到 List 頁）＋ `TsMenuAd`（`FAncestorId`/`FDescendantId`
  的 closure table，記選單樹的祖先-子孫關係，刪選單要連這張一起清）。
- `TsRelation`：兩個 Unit 之間的頁籤式關聯（`FUnitId1`/`FUnitId2`），例如母表單
  底下掛一個子功能的頁籤。
- `TsLocalApi`（`FTarget`=類別全名.方法, `FLocalApiGroupId`）＋
  `TsLocalApiGroup`：Bearer Token 的 LocalApi 端點註冊表，某些 integration
  class（不是走 `@WebServlet`）是靠這張表掛路由的，刪功能記得查。
- **`~lenuts*`／`~lzhcnts*` 開頭的表其實是 VIEW，不是實體表**——可以
  `SELECT` 查多語系標題，但 `DELETE FROM` 會報
  `The target table ... of the DELETE is not updatable`。這是已知現象（
  `ecp-lab2-instance` 那台機器的坑也記過同一件事，注意排除這些表當 orphan
  table 的假陽性，不要浪費時間硬刪）。
- **業務資料在 `Tc`/`Tp`/`Tw`/`Tt` 開頭的表**（`TsUnit.FTable` 指到的那張），
  跟 `Ts*` metadata 完全是两回事——**metadata 只有幾十筆，但業務資料表可能有
  幾十萬筆真實歷史紀錄**（實測一個 USB 控管稽核事件表有 14 萬筆）。使用者說
  「清 metadata」時，先分開回報這兩類的筆數，業務資料表要不要一起刪必須另外
  明確問過，不要預設含在「metadata」範圍內。

## 移除一整個功能模組的標準流程

1. `TsUnit` 用 `FActionClassName LIKE '%關鍵字%'` 找出所有相關 Unit 的 `FId`。
2. 逐一查 `TsPage`（拿 `FUnitId`）、`TsForm`/`TsList`（拿 `FFormId`/`FListId`）、
   `TsMenu`（`FPageId` 對應 `TsPage.FId`）、`TsRelation`（`FUnitId1`/`FUnitId2`）、
   `TsLocalApi`（`FTarget LIKE '%關鍵字%'`）。
3. 用 `information_schema.columns` 找出所有含 `FUnitId`/`FPageId`/`FFormId`/
   `FListId` 欄位的表（見上面清單，這個框架有上百張表有這些欄位，逐一
   `COUNT(*)` 篩出非零的才要處理，`Tc`/`Tp`/`Tw`/`Tt` 開頭的另外挑出來跟使用者
   確認）。
4. **刪之前一定先備份**：`mysqldump.exe` 對要刪的資料表下
   `--no-create-info --complete-insert --where="..."`（要 DROP TABLE 的話整張
   dump 含 schema）。備份檔留在遠端夠了，除非使用者要求，不用主動抓回本機
   （抓錯地方/多此一舉都會造成困擾，先問）。
5. 依 children→parents 順序刪（`TsListField`→`TsList`、`TsFormField`/
   `TsFieldGroup`→`TsForm`、`TsMenuAd`→`TsMenu`、最後才刪 `TsPage`/`TsUnit`）。
6. 刪完程式碼（`WEB-INF\classes\com\...` 底下對應的 package）記得先備份
   （`xcopy /e /i /q`），確認全系統沒有其他 `.class` 引用該 package
   （`findstr /s /m /c:關鍵字 *.class`，排除自己這個 package 內的命中）才刪。
7. 全部改完要**重啟 Tomcat**（見下）讓記憶體內的 metadata cache 跟資料庫/檔
   案系統同步——單純改資料庫、不重啟的話，正在跑的 process 可能還殘留舊的
   Unit/Menu/Form cache。
8. 收尾把過程中留在遠端的暫存 .sql / .bat / 排程工作都清掉。

## 重啟 Tomcat 的兩個陷阱

1. **`server.bat` 用相對路徑**（`apache-tomcat/bin/catalina run`），透過
   Task Scheduler 或 `Start-Process` 觸發時，如果沒有正確設定工作目錄，會直接
   報 `'apache-tomcat' is not recognized as an internal or external command`
   然後整個啟動失敗，不會有任何錯誤跳出來（因為是背景/detached 執行）。
   `Start-Process -WorkingDirectory` 這個參數在這台機器上透過 SSH 觸發時**不可靠
   **（實測失敗），穩定作法是先寫一個 wrapper .bat：
   ```bat
   cd /d C:\aipower
   call server.bat
   ```
   再用一次性排程工作觸發（比 `Start-Process` 穩，因為是全新 session 啟動，
   不受目前 SSH exec session 的工作目錄/環境影響）：
   ```bash
   plink ... "schtasks /create /tn TmpRestart /tr C:\aipower\restart_wrapper.bat /sc once /st 23:59 /ru SYSTEM /f"
   plink ... "schtasks /run /tn TmpRestart"
   # 等 java.exe 出現後：
   plink ... "schtasks /delete /tn TmpRestart /f"
   plink ... "del C:\aipower\restart_wrapper.bat"
   ```
2. **`server.xml` 沒設定 shutdown port**（`catalina stop` 會報
   `No shutdown port configured. Shut down server through OS signal.` 然後
   什麼都沒關掉）。要停掉現有行程只能 `taskkill /pid <pid> /f`（先用
   `tasklist /fi "imagename eq java.exe"` 找 pid，注意這台機器常常同時有人用
   RDP 登入手動開了另一個 java.exe，殺之前用 `Get-CimInstance Win32_Process`
   查 `CommandLine`/`Session Name` 確認是 Tomcat 而不是別人在跑的東西）。

啟動是否成功看 `C:\aipower\apache-tomcat\logs\catalina.<日期>.log` 的
`Server startup in [N] milliseconds`，並用 `findstr /c:"嚴重"` 掃有沒有
`Failed to initialize component`/`startup failed due to previous errors`
（這通常代表 port 被佔用——最常見原因是同時有另一個 java.exe process 在跑，
兩邊搶 22821/22822）。

## 除錯「連不上」抱怨的正確順序

使用者回報瀏覽器連不上（`ERR_CONNECTION_TIMED_OUT`）時，不要急著假設是伺服器
壞了，依序驗證，每一步都留證據：

1. `tasklist /fi "imagename eq java.exe"` + `netstat -ano | findstr :22821`——
   行程活著、port 有 LISTENING 在 `0.0.0.0`。
2. 主機**本機自己**打一次：
   `Invoke-WebRequest -Uri 'http://127.0.0.1:22821/aipower/xxx.page' -UseBasicParsing`
   ——排除「應用程式本身沒起來/context 部署失敗」。
3. 從操作端機器**跨網路**直接打同一個 URL（不是 127.0.0.1，是真實 IP）——
   排除「只有本機通、對外不通」（防火牆/routing 問題）。連續打 5 次看有沒有
   間歇性失敗。
4. `Get-NetFirewallRule -Direction Inbound -Enabled True | Where-Object {($_ |
   Get-NetFirewallPortFilter).LocalPort -eq '22821'}`——確認有 Allow 規則。
5. 如果 1-4 全部正常但使用者說還是連不上，很可能是**瀏覽器分頁卡住舊連線**
   （尤其如果剛好在同一時間重啟過 server），或使用者自己也在用 RDP 操作同一台
   機器、彼此的動作互相影響（判斷依據：`tasklist` 裡的 java.exe `Session Name`
   是 `Services` 還是 `RDP-Tcp#0`，加上 catalina log 的啟動時間戳記，可以分辨
   出「現在活著的這個 process 是誰、什麼時候起的」）。這種情況下請使用者
   Ctrl+Shift+R 強制重整或開新分頁再試，不要自己瞎猜著去改伺服器設定。
6. 如果透過 `claude-in-chrome` 工具本身連線不穩（回報
   `Browser extension is not connected`、或 `Frame with ID 0 is showing error
   page` 但緊接著又抓到成功的 network request），優先懷疑是**自動化工具的橋接
   不穩**，不要把這個雜訊當成目標伺服器真的有問題的證據。

---

## Conformance Addendum

## When to Use
Remote-manage the native (non-Docker) aipower/ECP install at 192.168.2.201 (hostname WIN11-1, domain "lab", port 22821/22822) over SSH — the OpenSSH client on this operator machine mangles `domain\user` logins and must be replaced with PuTTY's plink/pscp; the embedded MariaDB4j port changes every restart and must be discovered from the running mariadbd.exe process; and Tomcat's server.bat uses relative paths so any detached/scheduled-task restart must `cd` into C:\aipower first. Also covers the full procedure for permanently removing a business feature/module (code + DB metadata + DB business data) from a live instance, and how to debug "can't connect" complaints without chasing phantom server-side bugs. Use when the user asks to SSH into ".201", work on C:\aipower on 192.168.2.201, or mentions this host by IP/hostname. NOT `ecp-lab2-instance` (different machine, C:\Lab2, fixed port 3306 DB) or `aipower-docker-local` (Docker containers).

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
