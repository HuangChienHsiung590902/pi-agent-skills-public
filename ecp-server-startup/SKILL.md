---
name: ecp-server-startup
description: "啟動 Chainsea ECP（server.bat / aipower webapp）或獨立 MariaDB（database.bat）的完整流程與踩坑：Redis→Tomcat 啟動順序、embedded MariaDB「Data directory is not empty」重啟必炸的 bytecode patch、DB port 固定化、已知 schema bug、CDP Chrome 自動啟動。合併自原本的 ecp-server-startup-workflow 與 ecp-db 兩個 skill。"
triggers:
  - server.bat
  - redis 沒起來
  - timeout input redirection
  - Can't chdir to ./data
  - redis-server 沒有出現在程序列表
  - session manager failed to start
  - Data directory is not empty
  - Quicksilver startup failed
  - Failed to grow the connection pool
---

# ECP Server Startup（Redis→DB→Tomcat 完整啟動流程）

`server.bat` 啟動要依序搞定兩層完全不同的問題：**(A) Redis 必須先於 Tomcat 就緒**（否則 Session Manager 直接讓 webapp 部署失敗），**(B) embedded MariaDB 每次重啟都會因為資料夾非空而裝機失敗**（除非套用 bytecode patch）。這兩層互不相關但都會讓 `server.bat` 啟動失敗，排查時要先分清楚是哪一層。

## A. Redis → Tomcat 啟動順序

`server.bat` 必須依序啟動 **Redis → (delay) → Tomcat**。Redis 若未就緒，
Redisson Session Manager 會在 Tomcat deploy `aipower.xml` 時立刻拋出
`RedisConnectionException` 並使整個 webapp 部署失敗（不是 warning，是致命錯誤）。

### 三個非明顯的陷阱

1. **`timeout /t N > nul` 在 PowerShell `Start-Process` 啟動的 cmd 中會失敗**
   - 錯誤訊息：`ERROR: Input redirection is not supported, exiting the process immediately.`
   - 用 `ping 127.0.0.1 -n 3 > nul 2>&1` 取代（每次 ping ≈ 1 秒，n=3 ≈ 2 秒 delay）

2. **`redis-server.exe` 的 `dir` 設定是相對於 CWD，不是 conf 檔位置**
   - `redis.conf` 裡的 `dir ./data` 會相對於「誰呼叫 redis-server 時的工作目錄」
   - 正確做法：conf 用**絕對路徑** `dir C:\com\chainsea\redis\data`
   - 或在 server.bat 用 `pushd redis` / `popd` 讓 CWD 切進 redis\ 再啟動

3. **`start /B` 的 working directory 繼承自當前 cmd**
   - 在 server.bat 中，`pushd redis` → `start /B redis-server.exe redis.conf` → `popd`
     是最乾淨的寫法，不需要在命令列另外傳 `--dir`

### 判斷是不是這一層出問題

- Tomcat log 出現 `The session manager failed to start`
- Catalina log 出現 `Unable to connect to Redis server: 127.0.0.1/127.0.0.1:6379`
- `Get-Process redis-server` 沒有結果，或 redis-server 啟動後立刻消失
- Redis log 出現 `# Can't chdir to './data'`

### 診斷順序

1. `Get-Process redis-server` → 確認程序存在
2. `netstat -ano | findstr ":6379"` → 確認 port 在 listen
3. `redis-cli.exe ping` → 確認可連線
4. 看 `catalina.YYYY-MM-DD.log` 最後幾行有沒有 `Server startup in [N] milliseconds`（無錯誤）

### Current server.bat（已驗證可用）

```bat
cd /d "%~dp0"
set PATH=apache-tomcat/bin;%PATH%
set JAVA_HOME=jre
set CATALINA_HOME=apache-tomcat
set CATALINA_OPTS=-Xms256m -Xmx2048m -XX:MaxMetaspaceSize=512m -Dfile.encoding="UTF-8"

REM Start Redis in background (pushd ensures CWD is redis\)
pushd redis
start "Redis" /B redis-server.exe redis.conf
popd

REM Portable delay: ping localhost 3 times ≈ 2 sec
ping 127.0.0.1 -n 3 > nul 2>&1

REM Start Tomcat
apache-tomcat/bin/catalina run
```

### Key Files（Redis 層）

| 檔案 | 用途 |
|------|------|
| `chainsea/redis/redis.conf` | Redis 設定，`dir` 必須用絕對路徑 |
| `chainsea/redis/data/` | RDB 持久化資料目錄 |
| `chainsea/apache-tomcat/conf/Catalina/localhost/aipower.xml` | 啟用 Redisson Session Manager |
| `chainsea/apache-tomcat/conf/redisson.yaml` | Redis 連線設定（port 6379, database 10） |
| `C:\com\Kill ecp.ps1` | 關閉所有程序（java, mariadbd, redis-server） |

---

## B. Embedded MariaDB「重啟必炸」的 bytecode patch

- Persistent data dir: `C:\com\chainsea\mariadb\data`（真正的應用資料庫，**絕對不要刪**）。
- **正常跑法 = 直接 `C:\com\chainsea\server.bat`**（Tomcat）。`aipower` webapp 會**自己啟動一份 embedded MariaDB**，對著 `data` 目錄、綁一個自動挑選的空 port（不是 3306）。啟動後網址：
  `http://127.0.0.1:<httpPort>/aipower/`（httpPort 來自 `apache-tomcat\conf\server.xml`，這裡預設 22821）。登入：`administrator` / `111111`。
- `C:\com\chainsea\mariadb\database.bat` 是**獨立/維護用**的啟動器（在 port 3306 跑 MariaDB）。
  **不要**跟 `server.bat` 同時跑——兩個 mariadbd process 沒辦法同時鎖同一個 data 目錄。

### 關鍵：這台機器上有多份同一個 jar，要全部都補

每個 Tomcat webapp 有自己獨立的 `WEB-INF/lib`，所以 patch 必須套用到機器上**每一份** `mariaDB4j-core-*.jar`：

1. `C:\com\chainsea\tool\lib\mariaDB4j-core-*.jar` — `database.bat` 用的。
2. `C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\mariaDB4j-core-*.jar` — **`aipower` webapp**
   用的（這份才是 `server.bat` 需要的）。

只補 #1 會讓 `server.bat` 還是壞的。兩份都要補（以及任何其他找得到的 webapp 副本）。

### 這個 bug 的成因

`MariaLocalStarter.main` → `ch.vorburger.mariadb4j.DB.newEmbeddedDB()` → `DB.install()` 會
**無條件**執行 `mariadb-install-db.exe`。這支工具遇到非空的 `--datadir` 會直接拒絕：

```
mariadb-install-db.exe: ERROR : Data directory C:\com\chainsea\mariadb\data is not empty.
Only new or empty existing directories are accepted for --datadir
```

所以**第一次**跑沒事（空目錄），但之後**每次重啟都會失敗**（因為 `data` 已經有內容了）。
Chainsea 這層 wrapper 沒有做「已裝過就跳過」的判斷。

**修法：** patch `tool\lib\mariaDB4j-core-*.jar` 裡的 `DB.class`，讓 `install()` 具冪等性：

```java
void install() {
    if (new File(this.dataDir, "mysql").isDirectory()) {  // already initialized
        logger.info("Installation complete.");            // skip re-install
        return;
    }
    ... original install body (first-run empty-dir install still works) ...
}
```

這個 patch 要重算 exception table + StackMapTable（classfile 是 Java 17 / major 61）。
由本 skill 附帶的 `scripts/patch_install.py` 執行——一支純 Python 的 classfile 重寫工具，用
find-or-add 處理 constant-pool 項目，所以能適應不同 jar 版本的佈局。它是**冪等的**：
若 `install()` 已經被 patch 過（第一個 opcode 是 `new java/io/File`），就不會再動它。

### 附帶腳本二：scripts/patch_port.py（固定 embedded DB 監聽 port=3306）

`scripts/patch_port.py` 是另一支同類型的 bytecode patcher，補 `MariaLocalManager.class` 的 `config()`
方法：把 `iload_0`（port=0，代表隨機挑空 port）換成 `sipush 3306`（固定綁 3306）。同樣是
冪等的（印 `ALREADY PATCHED` 或 `PATCH WRITTEN`）。用途：當外部程序（例如 cbm-lite）需要用
固定 port 連進這個 embedded DB 時，用這支把隨機 port 行為關掉，改成每次都固定 3306。

```
python scripts/patch_port.py <jar_path>
```

正常情況（不需要外部程序固定連線）不用套這支，只有 `scripts/patch_install.py` 是每次啟動都建議跑的。

### 標準流程（正常啟動 app）

1. **補好每一份 `mariaDB4j-core` jar**（冪等，每次執行都安全）：

   ```powershell
   $py = "C:\Users\HCH\.claude\skills\ecp-server-startup\scripts/patch_install.py"
   Get-ChildItem -Recurse "C:\com\chainsea" -Filter "mariaDB4j-core-*.jar" |
     Where-Object { $_.Name -notlike "*.orig-bak" } |
     ForEach-Object { python $py $_.FullName }
   ```

   每一份會印 `ALREADY PATCHED`（no-op）或 `PATCH WRITTEN`，並備份成 `<jar>.orig-bak`。

2. **確認沒有獨立 DB 程序佔用 data 目錄**（不然 webapp 的 embedded mariadbd 會鎖不到）：

   ```powershell
   Get-CimInstance Win32_Process -Filter "Name='mariadbd.exe' OR Name='java.exe'" |
     Where-Object { $_.CommandLine -match 'MariaLocalStarter|database.bat' }
   ```
   跑之前先停掉這類程序（連同它的父 `cmd`/`java`）。

3. **啟動 app**：跑 `C:\com\chainsea\server.bat`（背景執行，log 導到例如
   `C:\Temp\server_run.log`），等 ~35 秒。

4. **驗證**（log + HTTP）：

   ```powershell
   Select-String C:\Temp\server_run.log -Pattern "Installation complete|Database startup complete|Quicksilver startup in|startup failed|Server startup in"
   ```
   成功標記：`Installation complete.`（裝機被跳過）→ `Database startup complete.` →
   `Quicksilver startup in N ms` → `Server startup in N ms`。接著：
   ```powershell
   $port = (Select-String C:\Temp\server_run.log -Pattern 'http-nio-(\d+)').Matches.Groups[1].Value
   Invoke-WebRequest "http://127.0.0.1:$port/aipower/" -UseBasicParsing   # expect 200
   ```
   若漏補某份 jar，失敗特徵是：`Quicksilver startup failed` + Atomikos
   `Failed to grow the connection pool` + `... data is not empty`。

### 獨立 DB 模式（維護用，選用）

跑 `C:\com\chainsea\mariadb\database.bat`（前景執行，MariaDB 在 port 3306，CTRL+C 停止）。
只能在 `server.bat` **沒在跑**的時候用。成功標記：`Installation complete.` →
`ready for connections.` → `MariaDB startup in N ms (port=3306)`；用
`Get-NetTCPConnection -LocalPort 3306 -State Listen` 驗證。

### 直接查詢正在跑的 embedded DB

webapp 的 embedded mariadbd **每次啟動都綁一個隨機空 port**（不是 3306）——從程序命令列找出來，
再用內附的 client 連（不用另外裝）：

```powershell
Get-CimInstance Win32_Process -Filter "Name='mariadbd.exe'" | Select-Object CommandLine
# ... --datadir=C:\com\chainsea\mariadb\data --port=61962 ...
```

```bash
"/c/com/chainsea/apache-tomcat/temp/MariaDB4j/base/bin/mariadb.exe" -h 127.0.0.1 -P <port> -u root default -e "SHOW TABLES;"
```

- 跑在 `--skip-grant-tables`，所以 `-u root` 不用密碼。
- Schema/database 名稱是 `default`（不是 `aipower`）。
- Client binary 在 `apache-tomcat\temp\MariaDB4j\base\bin\{mariadb,mysql}.exe`——這是
  MariaDB4j 啟動時才解壓出來的，所以只有 app 至少跑過一次之後才會存在。

### 已知資料庫 bug：`TcContact` 表不存在 → 「聯絡人」清單噴 SQL 錯誤

現象：打開 辦公自動化 > 聯絡人（或任何碰到 `TcContact` 的查詢）跳出 Aipower 錯誤對話框
「SQL執行失敗。語句：...from TcContact...」。真正的 driver 錯誤（要直接對 DB 跑該語句才看得到，
app 的錯誤對話框不會顯示）是：
```
ERROR 1146 (42S02): Table 'default.tccontact' doesn't exist
```

**根因：** `aipower-module-base-*.jar!/QS-MODULE/data/sql/default/init.sql` 定義 `TcContact` 時用了
兩個 `varchar(1000000)` 欄位（`FDuplicateContactIds`、`FChatSetDuplicateContactIds`）。在這個 DB 的
`utf8mb4` server charset（每字元 4 bytes）下超過 MariaDB 單欄位上限（utf8mb4 下約 16383 字元），
導致 `CREATE TABLE` 直接失敗：
```
ERROR 1074 (42000): Column length too big for column 'FDuplicateContactIds' (max = 16383); use BLOB or TEXT instead
```
`TsSqlUpgradeLog` 仍然會記錄 `init.sql` 已套用（runner 不會因為這一條語句就讓整份腳本失敗），
所以這個缺表很容易被忽略——upgrade log 裡完全看不出來。

**修法：** 從 jar 的 `init.sql` 抽出 `create table TcContact (...)` 那段，把那兩個欄位改成
`mediumtext`，再對正在跑的 DB 執行（安全——這張表本來就不存在，沒東西會被覆蓋）：
```bash
mkdir -p /tmp/x && cd /tmp/x && "/c/com/chainsea/jdk/bin/jar.exe" xf ".../aipower-module-base-*.jar" QS-MODULE/data/sql/default/init.sql
# extract the CREATE TABLE TcContact (...) block, sed the two varchar(1000000) -> mediumtext, then:
"/c/com/chainsea/apache-tomcat/temp/MariaDB4j/base/bin/mariadb.exe" -h 127.0.0.1 -P <port> -u root default < fixed_create.sql
```
若其他地方也出現「表不存在」錯誤，檢查該模組的 `init.sql`/升級用 `.sql` 有沒有一樣的
`varchar(1000000)` 寫法——這看起來是這個 schema/charset 組合下的系統性 vendor bug，不是單一事件。

### 已心（舊版）：搭配 server.bat 啟動時自動開 CDP Chrome

> ⚠️ **2026-07-24 更新：下面描述的 `open-chrome-debug.bat` 自動開 CDP Chrome 的做法屬於舊版
> `C:\com\chainsea` 安裝，那套安裝本身已不存在，對應的 MCP server 也已從 `~/.claude.json` 移除。
> 以下內容保留作為歷史參考（若之後其他 aipower 實例也想要一開啟就自動接 debug Chrome 的機制，可
> 從這裡取經，但**現在只用共用的 9222**，不要另建新 port）。目前共用的 `playwright-vrs` 固定接
> **9222**，見 `connect-chrome` skill。

`server.bat` 現在還會在 `catalina run` 之前跑 `start "" /min open-chrome-debug.bat`。那支腳本會輪詢
port `22821`，一旦 Tomcat 有回應，就開一個帶独立 debug port 的 Chrome（舊版的做法，實際上現
在應改用共用的 9222），指向 `http://127.0.0.1:22821/aipower`——所以每次 `server.bat` 啟動後不久
就有一個可 debug 的瀏覽器就緒。這台機器上還有另一個不明機制會在差不多時間開一個**非** debug 版的
普通 Chrome 視窗指向同個網址——無害，別跟 CDP 那個搞混。

### 備註

- **每次升級都要重補**：升級/重裝會整包覆蓋 jar，patch 是打在 jar 裡的，重裝就會消失。
  第 1 步是冪等的，重跑一次就好。
- 若 port 3306 已被佔用（`Get-NetTCPConnection -LocalPort 3306`），代表已經有一個 instance 在跑——
  不要再啟動第二個。
- 要復原：把 `<jar>.orig-bak` 複製回去蓋掉原本的 jar 即可。
- 相關背景記錄在 memory：`ecp-mariadb-install-patch`。

---

## Conformance Addendum

## When to Use
啟動 Chainsea ECP（server.bat / aipower webapp）或獨立 MariaDB（database.bat）的完整流程與踩坑：Redis→Tomcat 啟動順序、embedded MariaDB「Data directory is not empty」重啟必炸的 bytecode patch、DB port 固定化、已知 schema bug、CDP Chrome 自動啟動。合併自原本的 ecp-server-startup-workflow 與 ecp-db 兩個 skill。

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
