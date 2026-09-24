---
name: usql-crud
description: Use when running SQL CRUD queries (SELECT/INSERT/UPDATE/DELETE), inspecting table/schema structure, or copying data between different database engines from the command line via usql (D:\BIN\usql.exe) — a single universal CLI that talks to Postgres, MySQL/MariaDB, SQL Server, Oracle, SQLite, ClickHouse, Snowflake, BigQuery and 40+ other drivers with one consistent syntax.
---

# usql CRUD 操作

## 概觀

usql 是通用 SQL command-line 工具(`D:\BIN\usql.exe`,已在 PATH),用同一套語法連線並操作 40+ 種資料庫(Postgres/MySQL/SQL Server/Oracle/SQLite/ClickHouse/Snowflake/BigQuery...)。CRUD 就是標準 SQL 語句,透過 `-c`(單指令)或 `-f`(檔案)執行;`\d`/`\dt` 等 meta-command 用來查表結構;`\copy` 可跨資料庫搬資料。

## 連線前先掃本機有哪些 SQL server 在跑

在猜 DSN 之前,先確認本機到底有哪些資料庫服務可連。**優先用 port 掃描**——這台機器常見的 DB(例如 Lab2 的 MariaDB4j embedded 實例)是用 portable/embedded 方式跑,不會註冊成 Windows 服務,`Get-Service` 掃不到,只有監聽的 port 掃得到:

```powershell
$ports = @{1433='SQL Server'; 3306='MySQL/MariaDB'; 5432='PostgreSQL'; 1521='Oracle'; 6379='Redis(非SQL)'; 27017='MongoDB(非SQL)'; 9000='ClickHouse(native)'; 8123='ClickHouse(http)'}
foreach ($p in $ports.Keys) {
  $r = Get-NetTCPConnection -LocalPort $p -State Listen -ErrorAction SilentlyContinue
  if ($r) { "port $p ($($ports[$p])): LISTENING" }
}
```

再補查 Windows 服務(抓得到正規安裝、但抓不到 embedded/portable 實例,可交叉比對):

```powershell
Get-Service | Where-Object { $_.Name -match 'SQL|Maria|Postgre|Oracle' -or $_.DisplayName -match 'SQL|Maria|Postgre|Oracle' } | Select-Object Name, DisplayName, Status
```

- port 掃到就代表本機有對應的 server 正在跑,可以直接組 DSN(如 3306 → `usql my://user:pass@localhost:3306/dbname`)。
- 全部沒掃到 → 本機沒有可用的本地 SQL server,改連遠端(DSN 帶正確 host)或改用 `sqlite3://path.db` 這種免安裝的檔案型 DB。
- 每台機器情境不同(例如這台機器 3306 已被 Lab2 的 ECP/aipower embedded MariaDB 佔用,見專案記憶),掃到 port 後最好再用 `\l`(見下方)確認能不能真的連上、有哪些 database。

## 連線字串(DSN)

```
driver://user:password@host:port/dbname?param=value
```

常見 driver 別名(完整清單: `usql --no-init -c '\drivers'`):

| 別名 | driver |
|---|---|
| `pg`, `postgres` | PostgreSQL |
| `my`, `mysql`, `mariadb` | MySQL/MariaDB |
| `ms`, `mssql` | SQL Server |
| `or`, `oracle` | Oracle |
| `sq`, `sqlite` | SQLite3 |
| `ch`, `clickhouse` | ClickHouse |
| `sf`, `snowflake` | Snowflake |
| `bq`, `bigquery` | BigQuery |

SQLite 本機檔案範例:`sqlite3://path/to.db`(檔案不存在會自動建立)。

## 執行 CRUD

不進互動模式,直接用 `-c` 跑一或多個指令,依序執行、依序輸出:

```bash
usql -c "CREATE TABLE t(id INTEGER PRIMARY KEY, name TEXT);" \
     -c "INSERT INTO t(id,name) VALUES (1,'a'),(2,'b');" \
     -c "SELECT * FROM t;" \
     -c "UPDATE t SET name='updated' WHERE id=1;" \
     -c "DELETE FROM t WHERE id=2;" \
     'sqlite3://test.db'
```

- 多個 `-c` 依序在**同一連線**內執行,DSN 放最後一個參數。
- 大量 SQL 用 `-f script.sql` 讀檔執行。
- `-1`/`--single-transaction` 讓非互動執行整包包成單一 transaction(失敗全部回滾)。
- 互動模式:`usql 'DSN'` 進 REPL,SQL 語句用 `;` 結尾送出執行(等同 `\g`)。

## 查詢結構(Read schema)

| 指令 | 用途 |
|---|---|
| `\l` | 列出所有資料庫 |
| `\dt` | 列出所有 table |
| `\d NAME` | 顯示某個 table/view 的欄位結構 |
| `\di` | 列出 index |
| `\dv` | 列出 view |
| `\dn` | 列出 schema |
| `\ss TABLE` | 顯示該 table 統計資訊 |

範例:`usql --no-init -c '\dt' -c '\d t' 'sqlite3://test.db'`

## 跨資料庫搬資料(bulk Create)

`\copy SRC DST QUERY TABLE` 從來源 DSN 查詢結果,寫入目的 DSN 的 table,可跨不同 driver(例如 Postgres → MySQL)。指定目標欄位用 `TABLE(col1,col2,...)`。

## 輸出格式與批次選項

- `-J`(JSON)、`-C`(CSV)、`-H`(HTML)、`-A`(unaligned,適合管線後處理)、`-x`(expanded/vertical)。
- `-F sep`:自訂欄位分隔符(CSV/unaligned 模式)。
- `-o file`:結果輸出到檔案而非螢幕。
- `-q`:安靜模式,只印查詢結果,不印狀態訊息(適合 script 呼叫)。
- `-w`:不詢問密碼(DSN 已含密碼或用免密驗證時用)。
- `-X`/`--no-init`:跳過啟動腳本(測試/腳本呼叫建議加,避免載入個人 `.usqlrc`)。

## 常見錯誤

- 忘記把 DSN 放在所有 `-c` 之後 → usql 把它當成另一條指令解析失敗。
- SQLite 用 `sqlite3://` 前綴,不是裸路徑;相對路徑是相對「usql 執行時的工作目錄」,不是腳本檔位置。
- 互動模式打 SQL 忘記加 `;` 結尾 → 指令不會送出,卡在 buffer 等下一行輸入。

---

## Conformance Addendum

## When to Use
Use when running SQL CRUD queries (SELECT/INSERT/UPDATE/DELETE), inspecting table/schema structure, or copying data between different database engines from the command line via usql (D:\BIN\usql.exe) — a single universal CLI that talks to Postgres, MySQL/MariaDB, SQL Server, Oracle, SQLite, ClickHouse, Snowflake, BigQuery and 40+ other drivers with one consistent syntax.

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
