---
name: ecp-lab2-instance
description: C:\Lab2\chainsea 是目前真正在跑的 aipower 實例（port 22821，取代已損壞的 C:\Lab）。含：C:\Lab 損毀始末、官方安裝包全新安裝流程、已知安裝錯誤、固定 port 3306 獨立 DB 模式（讓 cbm-lite 等外部程序能連線）。用在：Lab2 需要重啟/重裝、要接外部程序到 Lab2 的 DB、或想搞懂「Lab/Lab2/com 三套 aipower 的關係」。
---

# C:\Lab2\chainsea — 目前的 live aipower 實例

## 現況（2026-07-04 起）

**`C:\Lab2\chainsea`（port 22821）是目前真正在跑、且 Cloudflare tunnel 對外服務的 aipower 實例**，取代了原本的 `C:\Lab\chainsea`（已損壞，保留不動當備份，見下）。
`hch.james-huang.org` 的主要路徑就是指到這台的 22821（見 `cloudflare-tunnel` skill）。

登入：`administrator` / `>*8ZvV1k`（安裝時隨機產生，**建議盡快改密碼**）。

> 相關：`ecp-deepseek`（在這台上自建的 DeepSeek 單元＋輕量客服台）、`cloudflare-tunnel`（對外路由）、`cbm-lite`（LINE 客服，現已整個搬進 `C:\Lab2\cbm-lite`）、`ecp-server-startup`（mariaDB4j 的 install-loop patch）。

## 一次啟動全部服務（2026-07-04 新增）

```bat
C:\Lab2\start-all.bat
```

依序：MariaDB(standalone,3306) → `chainsea\server.bat` 啟動的 Tomcat(22821) → 等 Tomcat 通了才啟動 `cbm-lite\start-cbm-lite.bat`(12621)。三個服務、四個 process（MariaLocalStarter + mariadbd + Tomcat java + CbmLiteServer java）。**沒有 Redis、沒有獨立 AiProxy**——那是舊版 `C:\com\chainsea` 才有的東西。

## C:\Lab 是怎麼壞掉的（教訓，別重蹈覆轍）

1. Lab 原本沒有裝 `aipower.crm` 模組（沒有服務諮詢/文字客服台的 unit/page 元資料）。
2. 為了裝起這個模組，用 `-Dqs.autoinit.enabled=true` 啟動一次（照 `ecp-server-startup`/memory 舊有做法），**框架本身的行為是「灌完 SQL 就自己關閉」**（一次性 install-then-shutdown 設計）。
3. 這個「灌完巨量 DDL（14000+ 筆）後立刻自我關閉」的動作，在這套 embedded MariaDB4j 環境下會造成 **InnoDB 資料字典來不及把新建的表持久化寫盤**——實體 `.frm`/`.ibd` 檔案都在，但 InnoDB 內部字典不認得它們（`Table 'x' doesn't exist in engine`）。
4. 結果：`aipower.crm` 模組建立的 **102 張表全部變成孤兒**（可用 `information_schema.TABLES` vs `information_schema.INNODB_SYS_TABLES` 比對抓出全部受損表，注意排除 `~lenu`/`~lzhcn` 開頭的多語系 VIEW 假陽性）。
5. 每次重啟，Registry 的「Complement unit relations」步驟一碰到任何一張孤兒表就整個 Quicksilver 啟動失敗（`Qs.Monitor.Sql` 會秀 `Table 'x' doesn't exist`）。
6. **教訓：不要對一個已經在跑的 aipower 用 `qs.autoinit.enabled=true` 追加安裝新模組。** 這個旗標是給「全新安裝」用的，追加安裝到已存在資料庫上會有此 race condition 風險。

`C:\Lab\chainsea\mariadb\data` 損壞現狀保留未刪，可當事後鑑識/備份。

## 修復方式：用官方安裝包全新安裝

不要嘗試手動修復 102 張孤兒表（風險高、不保證對齊官方 schema）。改用官方發行包對**全新資料夾**跑一次乾淨安裝（一般人的正常安裝流程，不會走到「追加模組」那條有 race condition 的路）。

發行包位置：`D:\ECP\OneDrive_2_2026-6-1\aipower-windows-7.3.12.5-*\`（同版本 7.3.12.5）。

### 安裝程式是互動式 CLI，用 heredoc 餵答案

```bash
cd "D:\ECP\...\aipower-windows-7.3.12.5-..."
printf 'C:\\Lab2\\chainsea\n\n\n\n\n\n\ny\n' | "jre/bin/java" -cp "lib/*" -Ddebug=false com.jeedsoft.quicksilver.toolset.pack.install.Install
```

問答順序（除第 1 步與最後確認外，其餘 Enter 用預設）：
1. Installation Path → 填目標路徑（例：`C:\Lab2\chainsea`）
2. Multi-tenant → 預設 n
3. Server Type → 預設 1（Tomcat）
4. HTTP Port → 預設 22821
5. HTTPS Port → 預設 22822
6. Modules → 預設 A（全部：quicksilver.main + aipower.base + aipower.crm + aipower.oa）
7. DataSource Configuration → 預設 `config/datasource.xml`
8. 最後確認 → **必須明確輸入 `y`**（空白/Enter 不算數，會一直重複問）

跑完會印出：`Administrator's password is >*8ZvV1k` —— 這組密碼要記下來。

### 已知安裝錯誤（4 個 SQL error，可忽略）

安裝結束會秀 `Install finished with 4 SQL error(s)`。詳細錯誤在**發行包自己的** `log\install-error.log`（不是裝到 Lab2 裡面，是安裝程式執行目錄下的 log）。

根因：`TcContact`（CRM 聯絡人表）欄位太多，單行超過 MariaDB InnoDB **8126 bytes** 的 row size 上限，導致 `create table TcContact` 失敗，後續對這張表的 `create index`/`alter table` 全部連鎖失敗（同一根因，非獨立問題）。

這是**安裝包本身既有的 bug**，不是這次操作造成的。`TcContact` 是聯絡人管理用表，跟服務諮詢/文字客服/DeepSeek 都無關，可以放著不管。

### 部署後必補的 mariaDB4j jar patch

跟任何新裝的 aipower 一樣，需要用 `ecp-server-startup` skill 的 `patch_install.py` patch 兩個 jar（否則每次重啟都因「Data directory is not empty」死循環）：

```powershell
$py = "C:\Users\HCH\.claude\skills\ecp-server-startup\patch_install.py"
python $py "C:\Lab2\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\mariaDB4j-core-3.1.0.jar"
python $py "C:\Lab2\chainsea\tool\lib\mariaDB4j-core-3.1.0.jar"
```

## 固定 port 3306 獨立 DB 模式（讓外部程序如 cbm-lite 能連線）

Lab2 預設用 `com.jeedsoft.marialocal.MariaLocalDriver`（embedded 模式）：Tomcat 自己內部管理 mariadbd，**每次啟動 port 都隨機**（`MariaDB4j\tmp\<random>`）。這對外部獨立程序（例如 cbm-lite 要直連 DB 讀 `TcChatMessage`）完全不可行。

### 改法：切成跟 production 一樣的「獨立固定 port」模式

1. 停 Lab2 Tomcat + 任何 embedded mariadbd。
2. 啟動獨立 `database.bat`（固定 port 3306，跟 `ecp-server-startup` skill 的 standalone 模式一樣，指向**同一個** `mariadb\data` 目錄，資料不用搬）：
   ```powershell
   Start-Process cmd.exe -ArgumentList "/c C:\Lab2\chainsea\mariadb\database.bat" -WorkingDirectory "C:\Lab2\chainsea\mariadb" -WindowStyle Hidden
   ```
3. 改 `C:\Lab2\chainsea\apache-tomcat\extension\aipower\config\datasource.xml`（注意：不是 WEB-INF/classes，是 `extension\aipower\config\`）：
   ```xml
   <driver-class>org.mariadb.jdbc.Driver</driver-class>
   <url>jdbc:mariadb://localhost:3306/default?useSSL=false&amp;characterEncoding=UTF-8</url>
   ```
   （原本是 `com.jeedsoft.marialocal.MariaLocalDriver` + `jdbc:marialocal:${root}/mariadb/data/default`）
4. 正常啟動 `server.bat`（這套的 `server.bat`很乾淨、不會自己砍 mariadbd，可以放心跟 standalone DB 同時存在）：
   ```powershell
   Start-Process "C:\Lab2\chainsea\server.bat" -WorkingDirectory "C:\Lab2\chainsea"
   ```
5. Tomcat 現在只是普通 JDBC client 連到已經在跑的 3306，**不會再自己起 embedded mariadbd、不會再隨機 port**。

改完之後，`cbm-lite.properties` 裡寫死的 `cbm.lite.db.url=jdbc:mariadb://localhost:3306/default...` 完全不用改，直接對得上。

### ⚠️ 重啟時的常見坑：殘留進程搶 port

多次重啟時很容易忘記殺乾淨舊的 `mariadbd.exe`/`java.exe`，導致新進程 `BindException: Address already in use` 而立即崩潰、卻誤以為「舊進程掛了」（其實是舊進程還活著、新進程根本沒真的起來）。**每次重啟前務必先確認乾淨**。

**不要**只用 `Get-Process mariadbd,java`（純粹比對行程名稱）——這台機器上同時存在 Lab2、AI3（`C:\ECP`）、測試用的 `C:\Aipower` 等多套實例，它們的 `mariadbd.exe`/`java.exe` 行程名稱完全一樣。用行程名稱查會被別套實例的行程誤導：明明 Lab2 自己乾乾淨淨，卻因為 AI3 或 C:\Aipower 正在跑而顯示「還有殘留」。

改用 `ExecutablePath`（Windows 解析出的絕對執行檔路徑，不受啟動時命令列寫相對或絕對路徑影響，天生跟安裝目錄綁定）只鎖定屬於 `C:\Lab2` 這套的行程：

```powershell
$installDir = "C:\Lab2"
$procs = Get-CimInstance Win32_Process | Where-Object {
    ($_.Name -eq 'java.exe' -or $_.Name -eq 'mariadbd.exe') -and
    $_.ExecutablePath -like "$installDir\*"
}
"remaining: $($procs.Count)"   # 應該是 0 才動手重啟
```

（這個踩坑跟修法在 `aipower-fresh-install` skill 裡有更完整的記錄，含一次用 `CommandLine` 誤判導致實際 `BindException` 故障的實測案例。）

## 原生「服務諮詢/文字客服」（Aiff/Vue）——裝起來了但很複雜，建議繞過

裝完整套 crm 模組後，`Aipower.ServiceConsult.Main`（`ecp/app/serviceconsult/user/main.html`）這個頁面終於存在了，但實測發現它是一整套 **Vue.js + 內嵌「Aiff」子平台**（`window.top.AiffJS.init()/.getProfile()/.getOpener()`），要有完整的 workgroup/queue/agent 狀態機才能正常顯示對話，直接塞測試資料進 `TcChatRoom`/`TcChatMessage` 不保證能被這層正確渲染。

**建議：不要死磕這條路。** 改用 `ecp-deepseek` skill 裡的「輕量客服台」（我們自己刻的簡單 JSP/JS，直接讀寫 `TcChatRoom`/`TcChatMessage`），或直接用 `cbm-lite` 的 `/agent` 主控台（更完整、已經跟 LINE 整合）。

## 選單注意事項

新增選單一律照 `ecp-menu` skill：掛在**子目錄底下**（例：辦公自動化→訊息管理→XXX），不可直接掛頂層根目錄。選單改動即時生效（F5 immediately），不需重啟；但新增 **Java 單元/Action** 一定要重啟 Tomcat。

---

## Conformance Addendum

## When to Use
C:\Lab2\chainsea 是目前真正在跑的 aipower 實例（port 22821，取代已損壞的 C:\Lab）。含：C:\Lab 損毀始末、官方安裝包全新安裝流程、已知安裝錯誤、固定 port 3306 獨立 DB 模式（讓 cbm-lite 等外部程序能連線）。用在：Lab2 需要重啟/重裝、要接外部程序到 Lab2 的 DB、或想搞懂「Lab/Lab2/com 三套 aipower 的關係」。

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
