---
name: aipower-docker-local
description: >-
  Manage the local D:\aipower Docker deployment (aipower-app + aipower-mariadb
  + aipower-pi-agent containers on this Windows machine's Docker Desktop;
  compose folder path has moved, verify with Test-Path first) — restart/
  relogin, reset admin password, clean up orphaned Unit/Page/Field/Form/List/
  Privilege metadata, menu-hiding field gotchas, building a standard six-core
  Unit whose Service layer calls an external HTTPS API, and compiling/
  deploying the hand-written LineChatConsoleServlet/LineWebhookServlet
  (/aipower/line/console, a raw HttpServlet outside the Unit/Page/Menu chain).
  Also covers the aipower-pi-agent container (pi-coding-agent + DeepSeek,
  exposed as "Pi 聊天室" at /aipower/pi/console) with full system control and
  MCP-based RAG wiring to a host LLM Wiki app. Use when `docker ps` shows
  aipower-app/aipower-mariadb/aipower-pi-agent. NOT the 10.145.119.19 remote
  instance or the native C:\com\chainsea / C:\Lab2 installs.
---

# aipower 本機 Docker 部署 (D:\Dockers\aipower)

> ⚠️ **2026-07-24 附註：下面歷史紀錄裡提到的獨立 CDP debug Chrome 設定已不存在**——那是舊版接
> `C:\com\chainsea`（非本 skill 管的 Docker 部署，而是另一套現已不存在的本機安裝）用的 `dev-aipower`
> skill 做法，那支 skill 與對應的 MCP server 已於 2026-07-24 一併刪除。**現在只用共用的 9222**
> （`playwright-vrs`，見 `connect-chrome` skill），不要再另外接其他 port 的 CDP。

> ⚠ **路徑更正（2026-07-19）**：這個 compose 專案的資料夾原本在 `D:\aipower`，後來被搬到/改名成 **`D:\Dockers\aipower`**（使用者自己重新整理 D 槽造成，非本 skill 操作導致）。`docker inspect aipower-app` 查出來的 compose label 仍然顯示建立當下的舊路徑 `D:\aipower`（容器建立後不會跟著更新），**不能拿 `docker inspect` 的 label 當作目前真正路徑**，這份文件裡所有 `D:\aipower\...` 路徑都已經改成正確的新路徑；如果又發現對不上，先用 `docker inspect aipower-app --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'` 查 label、再用 PowerShell `Test-Path` 直接驗證那個路徑是否真的存在，兩者不一致時以 `Test-Path` 為準（Bash/MSYS 的 `ls` 在這台機器上對這類路徑偶爾會失準，改用 PowerShell 的 `Test-Path`/`Get-ChildItem` 覆核）。

> ⚠⚠⚠ **2026-07-28：compose 專案路徑又搬回 `D:\aipower`**（不是上一條記載的
> `C:\aipower`）——`Test-Path 'D:\aipower\docker-compose.yml'` 為 `True`、
> `Test-Path 'C:\aipower\docker-compose.yml'` 為 `False`，容器內容跟 2026-07-26
> 那次記載的完全一致（`aipower-app`/`aipower-mariadb`/`aipower-pi-agent` 三個容器，
> 只有 `Ecp.Aile`/`CallLog`/`ChatAgent`/`ChatWorkGroup`/`DataTransfer`/`HelpDesk`/`Wf.*`
> 等官方內建 Unit，加上「Pi 聊天室」這個自訂功能），判斷是同一套環境被移動了資料夾，
> 不是又重建了一個新環境。**每次要動這個部署前，一律先用 `Test-Path` 現查一次，
> 不要相信任何一份記錄（含這份）寫死的路徑。**
>
> ⚠⚠ **2026-07-26：這是全新環境，不是同一套部署的延續！** 這天發現 compose 專案又搬到了 **`C:\aipower`**（`docker-compose.yml` 已確認在此，`aipower-app`/`aipower-mariadb` 兩個容器都是 `Up About an hour` 等級的新鮮貨），而且 **`aipower-app` 容器內 `WEB-INF/classes` 是空的、`web.xml` 只有一個 `<distributable/>`**——代表本文件裡記載的所有歷史整合（`LineIntegrationHome`/`LineChatConsoleServlet`/`AcdAdminServlet`/`AssetConsoleServlet`/`ChatReportServlet`/`AiProxy` 等等一整串 2026-07-18～25 的工作）**目前都不存在於這個容器裡**，即使原始碼還留在這個 skill 目錄下也一樣。**這代表 skill 記憶描述的是「曾經在某台機器上發生過的事」，不是「這個容器現在的狀態」**——每次要在既有基礎上加新功能之前，先用 `MSYS_NO_PATHCONV=1 docker exec aipower-app ls .../WEB-INF/classes` 或直接呼叫某個舊功能的 URL 確認那些歷史整合真的還在，不要預設它們在。這台機器當時也**完全沒裝 JDK**（`scripts/deploy-servlet.sh` 寫死的 `C:\Program Files\Microsoft\jdk-17.0.19.10-hotspot` 路徑找不到），用 `winget install --id Microsoft.OpenJDK.17 -e` 裝的同版本補上。見下面「Pi 聊天框」章節，那是在這個全新環境上從零開始建的第一個功能。

## 系統設計哲學（為什麼會這樣，不只是怎麼做）

Chainsea aipower（跑在 Quicksilver 框架上）是**元資料驅動（metadata-driven）**的平台：畫面本身不是程式碼，是資料。Java 只提供引擎（CRUD、事務、快取、Registry），「系統長什麼樣子」——有哪些功能、欄位、清單、表單、選單——全部存在 `Ts*` 開頭的資料表裡。下面幾點是這次踩坑後歸納出、會持續適用的推論，不是零散事實：

**三層鏈 Unit → Page → Menu，單向、缺一不可。**
- `TsUnit` 是最底層的「這個東西是什麼」：對應哪張 DB table、主鍵、以及五個 Java class 的完整類別路徑（Home/Dao/Service/Action/Api）。這是 Registry 在 Tomcat 啟動時把「資料庫的一張表」跟「Java 的一組物件」接線的**唯一依據**。
- `TsPage` 是畫面骨架，靠 `FUnitId` 指回某個 Unit，靠 `FActionMethodName='<UnitCode>.prepareList'` 這種字串去呼叫該 Unit 的 Action。它自己完全不含資料存取邏輯。
- `TsMenu` 是入口，靠 `FPageId` 指向某張 Page，**完全不認識 Unit 這個概念**。
- 這條鏈單向且不能跳：沒有 Page 的 Unit 上不了選單；沒有 Unit 的 Page 打不開（`XxxHome.getDao()` 回傳 null），**哪怕 Java 五層 class 檔案都乖乖躺在 jar 裡也一樣**——class 存在 ≠ Registry 有註冊，Registry 有沒有幫它注入 Dao 完全靠 `TsUnit` 那一筆 metadata。

**「安裝」在這套系統裡分兩層，而且常常只做一半。** 一個官方安裝包裡，Quicksilver 核心 + 系統管理這層 base 功能，跟 CRM 這種「業務模組」的資料，是**分開播種**的，而且播種本身不是原子操作——中斷在哪一步都可能留下「半套」孤兒資料（見下方孤兒清理章節）。這不代表部署壞了，是安裝腳本內部依賴關係設計如此，遇到「某個模組整組缺 `TsUnit` 但周邊 `TsPage`/`TsField` 甚至 `TsForm`/`TsList`/`TsPrivilege` 卻都在」不用意外，先當成已知模式處理。

**欄位存在不代表程式碼真的讀它。** `TsMenu` 底下一堆看起來像開關的欄位（`FEnabled`、`FLicensed`、`FHideInMainMenu`），但只有實際被某個 class/JS 引用的才有效（見下方選單隱藏章節，`FHideInMainMenu` 是死欄位、`FEnabled` 對 Administrator 角色被跳過）。每次要靠改一筆 metadata 改變系統行為，都得先確認「這欄位真的有程式碼路徑在讀」，不能只憑欄位命名合理就假設它生效——一定要在乾淨 session（`docker restart` + 走登入表單重新登入，見下方重啟章節）下實測驗證,不要憑感覺判斷有沒有生效。

## 環境資訊

- Compose 專案目錄：`D:\Dockers\aipower`（`docker-compose.yml` + `Dockerfile` + `entrypoint.sh`）
- Container：`aipower-app`（build 自官方 Linux 安裝包 `aipower-linux-7.3.12.5-...`，port 22821 HTTP / 22822 HTTPS）+ `aipower-mariadb`（`mariadb:lts`，host port `13306`→3306）
- DB：root / `<DB_PASSWORD>`，schema `default`
- 登入：`http://localhost:22821/aipower/` → `Administrator` / 密碼見下方（**不是** `111111` 的預設，也不是 10.145.119.19 那組 `<ECP_PASSWORD>` — 這是完全獨立的部署）
- `entrypoint.sh` 用互動安裝精靈但答案全空白，只完成了 Quicksilver 核心 + base 安裝；**CRM 業務模組（`TsModule` 表整個不存在）從未被安裝**，這是預期已知狀態，不是壞掉。
- 開機 log 會出現 `=== LicenseManager Initialize (Cracked) ===` — 安裝包本身帶的破解授權，如實記錄，非本 skill 造成。
- 備份腳本：`D:\Dockers\aipower\backup-restore.ps1`（單一專案用；若要跨所有本機 Docker，優先用 `D:\DockerBackups\docker-backup-menu.ps1`）
- **編譯部署腳本**：`scripts/deploy-servlet.sh`（這個 skill 目錄下）——編譯+部署任何手寫 Java servlet/六大核心 class 一律先用這支，不要手動重新推導 `javac -cp`，見下面「LINE 客服主控台」章節的「編譯與部署」子章節。

## 重啟與清快取（改 TsMenu/TsAccount 等設定後必讀）

**簡單重新整理網頁、甚至整個 `browser_navigate` 到新網址，都不會重建 session/menu 快取。** 唯一驗證有效的方法：

```bash
docker restart aipower-app
for i in $(seq 1 60); do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:22821/aipower/)
  [ "$code" = "200" ] && echo "UP after ${i}x2s" && break
  sleep 2
done
```

重啟後**務必用登入表單重新登入**（不要只是導到 MainFrame，會被踢回 `?jumpCode=SessionInvalid`，要走 `Qs.OnlineUser.Login.page`），否則改的東西一樣看不到效果——之前就是這樣白測了兩輪才發現問題出在快取而不是設定本身。

## 密碼重設（Docker 版）

`ecp-pwd` skill 的 `reset_password.py` 是針對**本機 embedded MariaDB**（自動偵測 `mariadbd.exe` process）設計的，這個 Docker 部署的 DB 在 container 裡、走 TCP 13306，那支腳本的 auto-detect 邏輯完全用不上。改用本目錄的 `scripts/reset_admin_password.ps1`：

```powershell
powershell -File "C:\Users\HCH\.claude\skills\aipower-docker-local\scripts/reset_admin_password.ps1"
# 或指定密碼/帳號/container
powershell -File ...\scripts/reset_admin_password.ps1 -Password "<EXAMPLE_PASSWORD>" -Login administrator -Container aipower-mariadb -DbPassword <DB_PASSWORD>
```

原理跟 `ecp-pwd` 一樣（`PasswordUtil`：8-byte salt + `"secret"` + 明文，SHA-256 疊代 1024 次，自訂 Base16 a-p 編碼），只是用 PowerShell 算 hash（這台機器 Git Bash 沒裝 python3）再透過 `docker exec ... mariadb` 寫入，不依賴本機行程偵測。

**⚠ 2026-07-19 教訓：登入失敗別無腦重試，記憶裡的密碼可能已經過期**。用記憶裡記錄的密碼連續登入失敗 2 次（`TsUserInputPasswordErrorCount.FCount` 逼近鎖定上限）才發現資料庫裡實際的雜湊根本不匹配那組密碼——不是加密流程有問題，是密碼真的在某個時間點變了但沒同步更新記憶。**正確順序**：連續失敗超過 1 次就先停手，查 `SELECT * FROM TsUserInputPasswordErrorCount WHERE FLoginName='administrator'` 看還剩幾次機會，優先直接跑 `scripts/reset_admin_password.ps1` 重置成已知密碼並清空鎖定計數，不要繼續用網路 API 賭「這次應該對了吧」。

## Unit / Page / Menu 架構速覽

完整規則見 `ecp-unit-setup-via-ui` skill。這裡只記這次踩過的重點：

- `TsUnit` = 資料引擎註冊表（DB table、Home/Dao/Service/Action class）；`TsPage` = 畫面骨架（List/Form/View 三選一，`FUnitId` 指回 Unit，`FActionMethodName='<UnitCode>.prepareList'`）；`TsMenu` = 選單入口（`FPageId` 指向 Page）。**單向鏈**：Unit → Page → Menu，`TsMenu` 完全不認識 Unit，只認識 Page；沒有 Page 的 Unit 上不了選單，沒有 Unit 的 Page 打不開（`XxxHome.getDao()` 回傳 null，即使 Home/Dao/Service/Action 這五層 class 檔都在 jar 裡也一樣——class 存在 ≠ Registry 有註冊）。
- 判斷「這個 Unit 是不是真的缺」：跟一份完整參考 dump（如另一台機器的 `default.sql`）比對 `TsUnit.FId`：
  ```bash
  grep -oP "(?<=INSERT INTO \`tsunit\` VALUES \(')[a-f0-9-]{36}(?=', ')" /path/to/reference/default.sql | sort > /tmp/sql_ids.txt
  docker exec <db_container> mariadb -uroot -p<pw> default -N -e "SELECT FId FROM TsUnit;" | sort > /tmp/live_ids.txt
  comm -23 /tmp/sql_ids.txt /tmp/live_ids.txt > /tmp/missing_ids.txt   # 存在參考庫、live DB 沒有 = 缺失
  ```

## 孤兒 metadata 清理（缺 Unit 但殘留 Page/Field/Form/List/Privilege...）

Unit 缺失本身不用管（它就是不存在，沒東西可刪）；真正要清的是**指向這些不存在 Unit ID 的殘留設定資料**。用 `scripts/sweep_orphan_unit_refs.sh`：

```bash
# 1. 先 dry-run 看全 schema 有哪些表、哪些 ID 還牽連著殘留（不會動資料）
bash scripts/sweep_orphan_unit_refs.sh aipower-mariadb root <DB_PASSWORD> default /tmp/missing_ids.txt

# 2. 確認清單合理後加 --apply 真的備份+刪除
bash scripts/sweep_orphan_unit_refs.sh aipower-mariadb root <DB_PASSWORD> default /tmp/missing_ids.txt --apply
```

腳本邏輯（已用兩批真實殘留資料驗證過，一批只有 TsPage+TsField 各 6 筆的輕量殘留，一批是 TsForm/TsFieldGroup/TsFormField/TsList/TsListField/TsPrivilege/TsQuerySchema/TsImportTemplate/TsKeywordField 全套都有、只差 TsUnit 這一筆的「幾乎完整半套 unit」）：

1. 掃 `information_schema.COLUMNS` 找出所有有 `FUnitId` 欄位的表（排除 `~l<lang><table>` i18n passthrough view）
2. 對每個表 `COUNT(*) WHERE FUnitId IN (...)`，一次印出全表報告（大部分表會是 0，`tc*`/`tp*`/`tt*` 業務資料表尤其該是 0——若不是 0 代表真的有人用過這個「幽靈」單元存過資料，要停下來跟使用者確認,不要自動刪）
3. `--apply` 時先處理「非 FUnitId 直連、要透過父表 ID 才能連到」的子表（`TsListField`→`FListId`、`TsFormField`/`TsFieldGroup`→`FFormId`、`TsRolePrivilege`→`FPrivilegeId`），逐表 `mariadb-dump --where` 備份到 `D:\aipower\backups\orphan_sweep_<table>_<timestamp>.sql`，再 `START TRANSACTION` 內依「子表先、父表後」順序 DELETE，最後複查歸零
4. 執行前一定會先查 `TsMenu.FPageId` 有沒有指到要刪的 `TsPage`——有的話要先處理選單，不要留死連結（目前兩次實測都是 0，但腳本每次都會查）

## 選單隱藏踩過的坑：Administrator 會跳過隱藏設定

想讓「系統管理」左側選單少幾項時，**兩個看似合理的欄位都測過、對 Administrator 帳號都無效**：

| 欄位 | 結果 | 原因 |
|---|---|---|
| `TsMenu.FHideInMainMenu` | 完全無效 | 整個 jar + 前端 JS 搜不到任何程式碼讀這個欄位——是廢欄位（至少 aipower 7.3.12.5 / quicksilver-module-main 7.2.2 這個版本組合是如此） |
| `TsMenu.FEnabled` | DB 端確認是真正被 `MenuServiceImpl` 使用的欄位，但改了、重啟容器、全新登入後 Administrator 帳號的選單樹依然完整顯示 | 系統管理員角色會直接跳過選單啟用/隱藏判斷，框架設計上讓最高權限角色永遠看得到完整配置樹，避免管理員把自己鎖在外面 |

**驗證方法一定要走「`docker restart` + 登入表單重新登入」**（見上面重啟章節），單純換頁測不出真假，之前就在這裡浪費了兩輪。

**結論**：如果操作帳號是 Administrator/系統管理員角色，選單層級的「隱藏」對它無效，只有兩條路：
1. 真的 `DELETE FROM TsMenu`（連帶檢查有沒有子選單、有沒有其他表引用 `FPageId`——但 `TsMenu` 本身通常不被其他表引用，直接刪安全），backup 方式同上面孤兒清理章節
2. 建一個非管理員角色的測試帳號，`FEnabled`/隱藏設定對它才會真的生效

兩者選一之前記得先問使用者要哪條路——刪 `TsMenu` 是真的動了 UI 結構,雖然可備份回復,但影響面比隱藏大。

## 登出/逾時後不會跳回登入畫面（跳去空白藍畫面）

**症狀**：登出、session 逾時、或直接開 `http://localhost:22821/aipower/`（無 session）都不會看到標準登入表單，而是卡在一片空白藍畫面，網頁標題是「Aiff登錄」；`document.body.innerHTML` 裡其實藏著一個 Vue `<template id="app">`，內容是給 LINE LIFF 用的客服登記表單（姓名/電話/性別），不是管理後台登入頁。

**根因**：`TsSystemParameter` 裡有一組互相矛盾的系統參數：

```sql
SELECT FKey, FValue FROM tssystemparameter WHERE FKey IN
  ('QsCustomLoginPageEnabled','QsCustomLoginPageUrl','AipowerEnableAile');
-- QsCustomLoginPageEnabled = 1                    <- 開了「自訂登入頁」
-- QsCustomLoginPageUrl     = Ecp.Aile.Login.page  <- 指向 Aile 客服登入頁
-- AipowerEnableAile        = 0                    <- 但 Aile 模組本身沒開！
```

`QsCustomLoginPageEnabled=1` 讓框架把所有「需要登入」的入口（含登出後跳轉、session 逾時跳轉、裸 root URL）都導去 `QsCustomLoginPageUrl` 指定的頁面，而不是標準的 `Qs.OnlineUser.Login.page`。這裡指到的 `Ecp.Aile.Login.page` 是給 LINE 客服場景用的 Aile 模組登入頁，但 `AipowerEnableAile=0`，Aile 後端沒開，Vue app 打的初始化 API 要嘛失敗要嘛卡住，畫面就停在空白（不是「Loading」文字，是整個看不到，可能跟 CSS 版面在沒有正確參數時的預設狀態有關）。這組參數矛盾應該是官方安裝包的預設值本來就沒對齊（不是我們哪個操作造成的）。

**修法**：關掉自訂登入頁開關，退回標準登入頁：

```bash
docker exec aipower-mariadb bash -c "mariadb-dump -uroot -p<DB_PASSWORD> default tssystemparameter --where=\"FKey='QsCustomLoginPageEnabled'\"" > D:/Dockers/aipower/backups/systemparam_QsCustomLoginPageEnabled_$(date +%Y%m%d_%H%M%S).sql
docker exec aipower-mariadb mariadb -uroot -p<DB_PASSWORD> default -e "UPDATE tssystemparameter SET FValue='0' WHERE FKey='QsCustomLoginPageEnabled';"
docker restart aipower-app   # 系統參數也走這層快取，一樣要重啟才生效
```

重啟＋重新整理後，裸 root URL / 登出 / session 逾時就會正確落在標準登入表單。已在這個部署上實測驗證通過。

## 建一個標準六大核心 Unit，Service 層呼叫外部 HTTPS API（2026-07-19 實測，Ecp.MikopbxExtension/Ecp.MikopbxCallLog/Ecp.MikopbxSetting）

需求範例：aipower 內建一個標準清單頁，開啟/按「重新整理」時，後端 Java 直接打外部系統的 REST API（本例是同機 Docker 跑的 MikoPBX），把資料同步進 aipower 自己的實體表。跟 `ecp-unit-design`/`ecp-unit-setup-via-ui` 的標準流程完全相容，只是 Service 層多一段 HTTP 呼叫邏輯。完整流程：

### 1. 六大核心怎麼寫（比對現成範例反編譯確認的介面）

用 `javap -p -cp <jar> <class>` 反編譯框架既有的小型範例 Unit（例如 `com.chainsea.ecp.chat.ChatOpenEntityHome` 一組）可以直接照抄介面規格，不用猜：
- `XxxHome`：`public static final UUID UNIT_ID = UUID.fromString("...")`；`getDao()`/`getService()` 各自呼叫 `Registry.getDao(UNIT_ID)`/`Registry.getService(UNIT_ID)` 再 cast。
- `XxxModel extends EntityModel`：4 個建構子照抄（無參、`Record`、`ResultSet`、`JSONObject`）；每個業務欄位的 getter/setter 都是薄封裝，呼叫繼承來的 `getString/getDate/...(欄位名)` 跟 `put(欄位名, 值)`——**Model 內部就是一個 `Map<String,Object>`**（`com.jeedsoft.common.advanced.db.dataset.Record`），沒有真正的 Java 欄位。
- `XxxDao extends EntityDao<XxxModel>` / `XxxDaoImpl extends EntityDaoImpl<XxxModel>`：純資料存取不用寫新方法，CRUD 全部繼承，**但 `XxxDaoImpl` 一定要寫一個明確呼叫 `super(UNIT_ID, XxxModel.class)` 的無參數建構子**，見下方 1.1 節,不能真的完全空殼。
- `XxxService extends EntityService<XxxModel>`：**在這裡宣告自訂方法**，例如 `void syncFromMikopbx(ServiceContext ctx);`。
- `XxxServiceImpl extends EntityServiceImpl<XxxModel>`：實作那個自訂方法，內部用 `ctx.getDaoContext()` 拿 `DaoContext`、`XxxHome.getDao()` 拿 Dao，直接呼叫 `dao.exists/insert/update(daoContext, id, model)` 寫資料。**同樣要寫明確呼叫 `super(UNIT_ID, XxxModel.class)` 的建構子**。
- `XxxAction extends EntityAction<XxxModel>` / `XxxActionImpl extends EntityActionImpl<XxxModel>`：**同步的真正觸發點**——見下方第 2 節，不要新建一顆自訂按鈕。**這個也要寫 `super(UNIT_ID, XxxModel.class)` 建構子**，三層都要寫,漏一層就漏一層的 Cast 例外。

### 1.1 ⚠ 致命坑：Dao/Service/Action 沒寫建構子＝Model 型別被寫死成基底類別，強轉必炸

`EntityDaoImpl<T>`（`EntityServiceImpl`/`EntityActionImpl` 同理）的**無參數建構子**原始碼反編譯出來是：

```java
public EntityDaoImpl() {
    super();
    this.modelClass = EntityModel.class;   // 寫死成基底類別，不是 null！
}

public EntityDaoImpl(UUID unitId, Class<T> modelClass) {
    super();
    this.modelClass = EntityModel.class;
    setUnitId(unitId);
    setModelClass(modelClass);   // 只有這個建構子才會設對
}
```

而 `getModelClass()` 的邏輯是「`modelClass` 是 `null` 才用反射自動推導真正子類別，不是 `null` 就直接回傳快取值」。因為無參數建構子已經把它設成 `EntityModel.class`（**不是** `null`），這個反射推導的分支**永遠不會被觸發**。

如果 `XxxDaoImpl`（含 Service/Action）只寫 `extends EntityDaoImpl<XxxModel> implements XxxDao {}` 這種真正的空殼、沒有明確寫建構子，Java 會自動幫你產生一個「什麼都不做」的無參數建構子，效果等同上面那個——`modelClass` 永遠停在基底 `EntityModel.class`，框架讀出每一列資料、要轉型成 `XxxModel` 時就會噴：

```
java.lang.ClassCastException: class com.jeedsoft.quicksilver.base.model.EntityModel
cannot be cast to class <你的 XxxModel>
```

**症狀特徵**：清單頁本身可能看起來正常（`getListData` 走的是別條路徑），但**點「打開」/雙擊開表單**（`prepareForm` → `getDataJson` → `getItem`）、或任何呼叫到 `dao.getItem/optItem/insert/update` 的地方就會炸——這是本次真實踩過的坑（2026-07-19，`Ecp.MikopbxExtension` 部署後點開分機記錄直接 500）。

**正確寫法**（反編譯框架真實案例 `ChatOpenEntityDaoImpl`/`ChatOpenEntityServiceImpl` 確認）：

```java
public class XxxDaoImpl extends EntityDaoImpl<XxxModel> implements XxxDao
{
    public XxxDaoImpl()
    {
        super(XxxHome.UNIT_ID, XxxModel.class);
    }
}
```

`XxxServiceImpl`/`XxxActionImpl` 完全比照辦理（`EntityServiceImpl`/`EntityActionImpl` 都有同樣一組雙建構子，同樣的坑）。**三層（Dao/Service/Action）都要寫，缺哪一層就是哪一層的操作會炸**，不是寫一層就全部安全。第九節那份「零 Java 純 metadata」骨架（class-name 四欄留 NULL）不受影響——這個坑只發生在「有寫 Java 六大核心」的情況。

### 2. ⚠ 別想用「清單頁自訂按鈕」觸發自訂 Action 方法——這條路在這個框架版本走不通

反編譯共用樣板 `quicksilver/page/template/EntityList.jsp`，它只固定載入三支 JS（`CommonBusiness.js`/`QueryForm.js`/`EntityList.js`），**沒有任何機制載入 per-unit 自訂 JS**。資料庫裡看得到的其他 Unit 有自訂按鈕（如 `DepartmentList.doEnable`）也是走標準 `EntityList.jsp`（同一支模板、`FLoadHandler` 是 NULL），代表這些自訂 JS 命名空間（`DepartmentList` 之類）一定是從別的、未查清楚的載入機制來的（可能是某種依 Unit/Module 綁定的全域 bundle），**不值得花時間逆向**。

**正確做法：把同步邏輯直接寫進 `ActionImpl.prepareList()`，覆蓋掉繼承來的版本**：

```java
@Override
public PageResult prepareList(ActionContext ctx)
{
    try {
        XxxHome.getService().syncFromMikopbx(ctx.getServiceContext());
    } catch (RuntimeException e) {
        logger.warn("sync failed, showing cached data", e); // 外部 API 掛了不要讓整頁掛掉
    }
    return super.prepareList(ctx);
}
```

這樣**清單頁自帶的標準「重新整理」按鈕**（`EntityList.doRefresh` → 再呼叫一次 `prepareList`）天生就會觸發同步，不用額外接線，完全用已驗證存在的機制做到「手動按鈕重新整理」的需求。額外保留一個可直接呼叫的 `syncFromMikopbx(ActionContext ctx)` 方法（回傳 `DataResult`）方便之後不透過 UI、直接 `curl POST /aipower/Ecp.Xxx.syncFromMikopbx.data` 測試或整合。

### 3. 呼叫外部 HTTPS API（自簽憑證）的兩個連續坑

用 `java.net.http.HttpClient`（JDK 內建，不用加套件；JSON 解析用 classpath 裡本來就有的 `gson-2.9.0.jar`，不用另外加 lib）打外部服務時，如果對方是同機 Docker、自簽憑證，會連續踩兩層：

1. **HTTP 明碼請求被 301 導去 HTTPS**：如果對方 nginx 設定「Host header 不是預期值就強制轉 HTTPS」，直接對 HTTP port 發請求會拿到 301，body 是 nginx 預設頁不是 API JSON——**必須直接打 HTTPS**，不要指望跟著 redirect（Java `HttpClient` 預設會跟 redirect，但轉址位址、port 常常對不上容器對外映射，不可靠）。
2. **HTTPS 自簽憑證：光是信任憑證鏈還不夠，hostname 驗證是獨立的第二關**。`SSLContext` 配一個信任全部憑證的 `TrustManager`只解決「憑證鏈可信任」，Java `HttpClient` 還會另外拿連線用的 hostname 去比對憑證的 Subject Alternative Name（SAN），對不上一樣丟 `SSLHandshakeException: No subject alternative DNS name matching`。**用 `HttpClient.Builder.sslParameters(SSLParameters)` 設 `setEndpointIdentificationAlgorithm("")` 想要整個關掉 hostname 驗證，實測在這個 JDK 17.0.19/HttpClient 組合下沒有生效**（原因不明，可能是 JDK 已知行為，見 JDK-8236039 這類討論——不要花更多時間在這條路上）。**真正可靠的做法：查出憑證的實際 SAN**（`echo | openssl s_client -connect <ip>:<port> | openssl x509 -noout -text | grep -A2 "Subject Alternative Name"`），**在呼叫端容器的 `/etc/hosts` 裡加一條把那個 SAN 名字對應到目標服務的真實 IP**，讓 URL 裡用的 hostname 字串直接等於憑證的 SAN，hostname 驗證就會自然通過，完全不用關掉驗證這種取巧手法。

    - 目標服務如果是 Docker Desktop 管理的另一個容器、只暴露給 host，用 `host.docker.internal` 解析到的 IP（`getent hosts host.docker.internal` 查，這台是 `192.168.65.254`，Docker Desktop 標準主機閘道）當 `/etc/hosts` 裡新增那行的 IP。
    - `docker exec <container> sh -c "echo '<ip> <cert-san-hostname>' >> /etc/hosts"` 可以立即生效測試，**但這條手改的 /etc/hosts 只在容器目前這條命活著時有效**——`docker restart` 就會被重置清空（親測），必須重加。要讓它永久生效，得改 `docker-compose.yml` 該 service 加 `extra_hosts:`，然後 `docker compose up -d`（**重新建容器**，比 `docker restart` 動作更大，套用前記得跟使用者確認時機）。
    - **對外目標服務的 port 對照文件常常是容器內部 port，不是宿主機實際映射 port**——一律用 `docker ps`/`docker inspect` 查真正對外的 port，不要照抄對方 API 文件寫的預設 port。

### 4. 部署（這個 Docker 專案沒有 `tool\src`，跟 C:\Lab\chainsea 那種原生安裝的 Java 開發流程不一樣）

**2026-07-21 起優先用 `scripts/deploy-servlet.sh`**（見下面「LINE 客服主控台」章節的「編譯與部署」子章節）——
六大核心的 Home/Model/Dao/DaoImpl/Service/ServiceImpl/Action/ActionImpl 這幾個檔案可以一次全部
當參數丟給它，它會照各自的 `package` 宣告自動算出容器內路徑分別部署，不用再手動組 `javac -cp`。
以下是背後原理與該腳本出現前的手動流程，仍值得了解：

1. `docker cp aipower-app:/app/apache-tomcat/webapps/aipower/WEB-INF/lib <本機暫存資料夾>` 整包複製出來一次當 classpath（100+ 個 jar，一次性，之後編譯都共用）。
2. 本機 `javac -cp "<暫存lib>/*" -d classes <六大核心 .java 檔案...>`（這台機器用 `C:\Program Files\Microsoft\jdk-17.0.19.10-hotspot\bin\javac.exe`，跟容器內 Temurin 17.0.12 bytecode 相容）。
3. `docker cp classes/com aipower-app:/app/apache-tomcat/webapps/aipower/WEB-INF/classes/`（這個目錄一開始不存在，`docker cp` 會自動建立；改完 Service 層邏輯之後**只需要 `docker cp` 覆蓋、不用重新 SQL**，重啟一次就會抓到新 class）。
4. Token/Base URL 這類設定值：仿 `application.properties` 的做法另開一個獨立 properties 檔（例：`mikopbx-integration.properties`，跟 `application.properties` 放同一個 `extension/aipower/config/` 目錄），Service 層用 `java.io.FileInputStream` 讀，**改這個檔案不用重啟**（每次呼叫都重新讀檔，沒有快取）——比研究 `TsSystemParameter` 的 Java 讀取 API（這個框架版本沒找到現成好用的）風險低、部署也更快。
5. `docker restart aipower-app`，用登入表單重新登入（見上面重啟章節），開清單頁驗證；查 `docker logs aipower-app | grep "Register unit: Ecp.Xxx"` 確認 Registry 真的載入了新 Unit。

### 5. ⚠ 驗證外部 API 資料真的有寫入資料庫，不要只看回應 `success:true` 就結案

`syncFromMikopbx()` 這類「呼叫外部 API → 寫入本地表」的方法，兩個地方特別容易靜默失敗（不拋例外、不報錯，但實際上什麼都沒做）：

1. **外部 API 回應結構理解錯誤**（2026-07-19 實測踩過）：`data` 底下不一定是扁平陣列——像 CDR/日誌這類「一組事件底下可能有多筆明細」的資源，很可能是**巢狀分組**（`data.records[].records[]`），判斷唯一性用的 ID 欄位往往只存在最內層。如果程式碼只解析到外層，抓到的物件會缺欄位（例如缺 `UNIQUEID`），而「缺關鍵欄位就 `continue` 跳過」這種防禦寫法會讓每一筆都被無聲跳過，回應照樣是 `{"success":true}`。**呼叫任何不熟悉的外部 API 前，先完整印出一次真實回應（別只信官方文件的簡化範例），逐層確認欄位在哪一層。**
2. **日期格式解析用錯 parser**：`OffsetDateTime.parse()` 要求標準 ISO-8601（帶 `T`、帶時區），但很多系統（含 MikoPBX）給的是 `"2026-07-18 08:52:33.583"` 這種空格分隔、無時區格式，直接丟進去會拋 `DateTimeParseException`，如果外層有 catch 就會靜默變成 `null`——資料照樣寫得進去，只是時間欄位永遠是空的。改用 `LocalDateTime` + 自訂 `DateTimeFormatter` pattern，並手動套上來源系統已知的時區。

**寫完 Service 層邏輯後，一定要做這一步**：實際觸發一次同步、直接查資料庫確認真的寫入且每個欄位都不是非預期的 `NULL`，不能只看 API 呼叫回傳 `success:true` 就當作沒事——這種「回應正常但資料是空的」模式沒有任何錯誤訊號會主動提醒你去檢查。

3. **不是 bug，是時間差——「剛掛斷馬上按重新整理」抓不到最新那一筆是正常現象**（2026-07-19 實測確認）：`Ecp.MikopbxCallLog` 的同步是「開清單頁/按重新整理」當下才觸發一次 API 呼叫，直接抓 MikoPBX `/pbxcore/api/v3/cdr` 當時的快照。如果使用者在通話**剛掛斷的瞬間**就按重新整理，MikoPBX 自己那邊的 CDR 落地（Asterisk 把通話明細寫進它自己的資料庫）可能還沒完成，那次同步抓到的清單自然不含最新這一筆——**不是程式碼漏抓，單純是「查詢時間點」早於「來源系統資料真正寫入的時間點」**。判斷方法：直接查 aipower 本地表（`SELECT * FROM TcMikopbxCallLog`）比對 MikoPBX 自己 CDR API 當下回傳的筆數，如果本地永遠只差最新一筆、其他都同步得到，等個幾秒再重新觸發一次同步通常就能補上，不用懷疑程式碼壞掉。

### 6. 寫回外部系統（PATCH/PUT）：override `doUpdate()`，不要 override `update()`（2026-07-19 實測，Ecp.MikopbxSetting）

前面 1~5 節都是「讀」方向（外部 API → 本地表）。要做「寫」方向（表單保存 → 外部 API → 本地表）時：

- **反編譯確認框架標準 CRUD 的真正落地點**：`javap -c -p` 反編譯 `EntityServiceImpl` 會看到 `public void update(ctx,id,model)` 內部依序做 checkExists→fill→checkPrivilege→validate→觸發Before事件→**`protected void doUpdate(ctx,id,model)`**（真正寫DB的地方）→觸發After事件→寫業務日誌→建全文索引。**Override `doUpdate()` 而不是整個 `update()`**，才能保留框架所有標準前置/後置行為，只在真正落地寫DB前插入外部呼叫邏輯。
- **寫回順序要保證兩邊一致**：`doUpdate()` 裡先呼叫外部 API，**成功才呼叫 `super.doUpdate(ctx,id,model)`**；外部呼叫失敗就直接拋例外、完全不寫本地表。避免「本地顯示已存好，但外部系統實際沒套用」的假象。用「暫時改錯 API token 模擬失敗」驗證過這個保護機制確實有效（本地表完全沒被寫入、`FLastSyncTime` 沒變）。
- **同步（讀）邏輯要繞開這個 override**：`syncFromMikopbx()` 裡寫本地表要直接呼叫 `dao.insert()`/`dao.update()`，不要呼叫 `this.update()`——否則「同步一次」會反過來觸發一次「PATCH 回外部系統」，變成荒謬的自我寫回迴圈。
- **前端表單保存的真正 API 端點跟 payload 格式，也要反編譯確認**：`EntityActionImpl.save(ActionContext)` 從 `ctx.getArguments().getObjectArray("data", modelClass)` 讀取，**payload 是 `{"data": [{...}]}` 陣列包物件**，不是單一物件；`FId` 有值就走 `EntityService.update()`，沒有就走 `create()`。前端表單本來就會把整個表單所有欄位目前值都提交（不只是被改的那一個），寫測試腳本驗證時也要照做，不能只送被改的那個欄位，否則其他整數/布林欄位會被沒帶到的值蓋成預設值 0。實際可呼叫的 URL 是 `Ecp.Xxx.save.data`，不是直覺以為的 `update.data`。
- **⚠ 別只信原始碼分析，一定要實測**：讀了 MikoPBX PHP 原始碼（`SaveSettingsAction.php`）以為存檔只寫 DB、不會觸發 Asterisk reload，所以一開始把 SQL 欄位警示文字寫成「需重啟才生效」。**實際拿 SIPPort 存檔測試，`asterisk -rx "pjsip show transports"` 幾秒內就顯示監聽 port 真的變了**——代表 MikoPBX 有背景機制會偵測設定表變化並自動套用，不需要重啟，但也不是瞬間生效（數秒延遲）。這推翻了原始碼分析的結論，說明「讀原始碼」跟「跑一次看真實行為」可能得出不同結論，兩者都要做，且**以實測為準**。任何會影響正在運作服務的欄位，測試前務必跟使用者確認「現在沒人在用」，測完立刻改回原值並用系統本身的診斷指令（這裡是 `pjsip show transports`）交叉確認真的恢復了。

## LINE 客服主控台：自訂 HttpServlet，不是 Quicksilver Unit（`/aipower/line/console`）

跟前面「六大核心 Unit」那套完全不同的另一種擴充方式：`com.chainsea.ecp.lineintegration` package 底下是一組**手寫的原生 `HttpServlet`**（`LineChatConsoleServlet`/`LineWebhookServlet`），由 `LineIntegrationStartupListener`（`ServletContextListener`）在 Tomcat 啟動時**動態註冊**，完全繞過 `TsUnit`/`TsPage`/`TsMenu` 那條標準鏈，也不呼叫原廠 `ChatAsdServiceImpl` 佇列引擎，直接讀寫原生 `TcChatRoom`/`TcChatMessage` 表。左側選單「開發→LINE 聊天框」只是嵌一個 `<iframe>` 指到這個 servlet 的網址，本身不是 Unit/Page。

- **`LineChatConsoleServlet`**（掛在 `/aipower/line/console`，需要一般 ECP 登入 session）：客服人員用的主控台，左側房間清單＋右側對話串。GET `/console`（HTML 本體，整包字串常數寫死在 `PAGE_HTML`）、`/console/rooms`、`/console/messages?roomId=`、POST `/console/send`、POST `/console/close`。
- **`LineWebhookServlet`**：LINE 平台 webhook 的匿名進入點（不需要登入 session），與此次改動無關。
- **`LineIntegrationHome`**：共用的資料庫連線/LINE Push API 靜態方法（`getDbConnection`/`pushLine`/`insertNativeChatMessage`/`closeNativeChatRoom` 等），兩支 servlet 共用。
- **原始碼永久保存位置**：`C:\Users\HCH\.claude\skills\aipower-docker-local\line-integration\src\`（含全部 `com.chainsea.ecp.line*` package）。**這份原始碼不在容器裡、也不在任何 `tool\src`，只有這裡跟已編譯的 class 檔（容器內 `WEB-INF/classes`）是唯一副本**——改動前一定要先從這裡讀最新版，改完也要把新版存回這裡，不要留在 session 暫存目錄（`/scratchpad/`），那裡會被系統清掉。

### 編譯與部署：用 `scripts/deploy-servlet.sh`，不要手動重新推導 classpath

**2026-07-21 起，編譯部署一律用這個 skill 目錄下的 `scripts/deploy-servlet.sh`**，不要再手動組 `javac -cp` 指令——這支腳本是把 2026-07-19～21 這幾次部署反覆踩出來的坑（`full-lib` 要快取、`deployed-classes` 每次要刷新、Git Bash 傳給原生 `javac.exe` 的 classpath 必須先用 `cygpath -w` 轉成 `C:\...` 格式否則所有 import 都報 `package does not exist`、部署前要查 `open_rooms` 確認沒有真人在用再重啟）一次寫死進腳本：

```bash
cd "C:\Users\HCH\.claude\skills\aipower-docker-local"
./scripts/deploy-servlet.sh line-integration/src/com/chainsea/ecp/lineintegration/LineChatConsoleServlet.java
# 多個檔案（含跨 package）一次給：
./scripts/deploy-servlet.sh acd-integration/src/com/chainsea/ecp/acd/AcdAdminServlet.java acd-integration/src/com/chainsea/ecp/acd/ChatReportServlet.java
# 只想編譯+複製、不想現在重啟（例如要等真人對話結束）：
./scripts/deploy-servlet.sh --no-restart <file.java>
# 已確認沒人在用、跳過 open_rooms 檢查直接重啟：
./scripts/deploy-servlet.sh --force <file.java>
```

行為：自動判斷每個檔案的 `package` 宣告推算容器內 `.class` 路徑（不用手動拼路徑）、部署前備份舊 class 成 `.bak-<timestamp>`、預設會查 DB 目前有沒有進行中的房間，有的話直接擋下來並印出提示（不會自己互動詢問，因為這支腳本可能在非互動環境跑），要嘛 `--force` 硬重啟、要嘛 `--no-restart` 先部署晚點再重啟、要嘛等對話結束再重跑。重啟後自動 curl 直到 200、並掃一次 `docker logs` 排除已知背景雜訊（`ServiceRequestUpgrade`/`TaskUpgradeRoutine`/`TimedEmailRoutine`/`TaskIndicatorsRoutine`/`ChatRoomHome.getService() is null`）後印出真正新增的錯誤。

`full-lib`（~170 個 jar）快取在 `<skill目錄>/.build-cache/full-lib`，**只在第一次執行時抓一次，之後長期重用**（跟舊版每次都要求手動 `docker cp` 一次性抽出來的做法不同，這裡是永久保存在 skill 目錄，不是 session scratchpad，跨對話都還在）；`deployed-classes` 則是**每次執行都重新抓**（成本低，確保編譯時參照到的相依符號跟容器內實際在跑的版本一致，不會因為忘記手動更新而編譯出跟線上對不上的東西）。

### 2026-07-20 改動：房間清單加「移除」按鈕

需求：客服台左側房間清單裡，已結束的對話會一直留著（`handleRooms()` 的 SQL 條件是 `FCloseTime IS NULL OR FCloseTime > NOW()-24小時`，24 小時內都會留在清單），使用者想要能自己把看過的舊房間從清單關掉。

**做法：純前端 `localStorage` 隱藏，完全不動資料庫**：
- `PAGE_HTML` 的 CSS 加了 `.room .dismiss`（每個房間項目右上角一個小 `×`）。
- JS 加了 `loadDismissed()`/`saveDismissed()`（讀寫 `localStorage['lineConsoleDismissed']`，存房間 ID 陣列）與 `dismissRoom(id)`：把 id 加進這個 Set、若正是目前開啟的房間就順便清空右側對話框，然後重繪清單時跳過已 dismiss 的 id。
- 三秒一次的 `setInterval(loadRooms,...)` 輪詢也會套用同一個過濾，所以不會被自動刷新蓋回來。
- **刻意不做的方案**：沒有幫 `TcChatRoom` 加 `FHidden` 之類的新欄位（這是原生共用表，加欄位風險較高、也要改 SQL/DAO 三處）；也沒有把 `FCloseTime` 往前偽造來借用現有 24 小時過濾（會污染其他報表可能用到的真實時間）。純前端方案的取捨：**只在同一台瀏覽器/同一個人有效，換瀏覽器或清 `localStorage` 又會全部跑出來**，是目前接受的限制，需求變成「所有客服共享同一份隱藏清單」的話才需要真的加 DB 欄位。
- 驗證方式：用 Playwright 對 `.room .dismiss` 按鈕 `.click()`（用 `browser_evaluate`，不要用 `browser_click`+ref——這頁面每 3 秒 `setInterval` 重繪一次 DOM，ref 抓到後常常來不及點就失效，見下方「已知踩坑」）。

### 2026-07-20 改動：抓目前登入帳號 + 就緒/未就緒狀態（為後續進線分派鋪路）

需求：主控台要能顯示「現在是誰在操作」，並讓客服自己標記就緒/未就緒，方便之後接「新進線只派給就緒客服」這類分派邏輯（分派邏輯本身這次沒做，只做狀態本身）。

**怎麼從一支 raw HttpServlet 拿到 ECP 目前登入帳號（不是六大核心 Action，沒有 ActionContext 自動注入）：**
反編譯 `quicksilver-module-main-7.2.2.jar` 的 `ContextFilter$InnerFilter.filter()` 找到框架自己怎麼做——`OnlineUserHome.getService().getItem(ac.getSession())` 拿 `OnlineUserModel`，再 `ActionContext.setThreadInstance(ac)`。**不用碰 `ActionContext.getThreadInstance()` 那條 ThreadLocal**（依賴 `ContextFilter` 的 url-pattern 有沒有覆蓋到 `/line/console/*`，沒查證前不可靠），直接照抄前半段、繞過 filter 自己呼叫同一個官方 API 最省事可靠：

```java
HttpSession session = req.getSession(false);
OnlineUserModel ou = session != null ? OnlineUserHome.getService().getItem(session) : null;
// ou.getAccountId() / ou.getAccountName() / ou.getUserName()（顯示名優先用 userName，空的話退回 accountName）
```

包成 `LineIntegrationHome.getCurrentOnlineUser(HttpServletRequest)` 靜態方法，未登入或 session 失效回傳 `null`。

**就緒/未就緒狀態儲存**：`LineIntegrationHome` 新增 `Map<UUID,Boolean> agentReadyStatus`（`ConcurrentHashMap`，純記憶體，按 `accountId` 存）。跟房間隱藏那次一樣的取捨邏輯：**這是在線狀態、不是業務資料**，Tomcat 重啟清空是預期行為（比照 ECP 原生 `OnlineUser` 概念），不值得為了「重啟後還記得」開一張表。

**新增兩個 servlet 路由**：
- `GET /aipower/line/console/whoami` → `{accountId, accountName, displayName, ready}`，未登入回 401。
- `POST /aipower/line/console/ready` `{ready:true|false}` → 寫入目前帳號的狀態並回傳確認值，未登入回 401。

**UI**：`PAGE_HTML` 最上面新增一條 `#agentBar`（深色列），左邊「👤 顯示名稱」、右邊一顆就緒/未就緒切換鈕（灰底未就緒／綠底就緒，跟送出按鈕同色系一致）。頁面結構從單純 `display:flex` 改成 `flex-direction:column` 外層＋原本 `#rooms`/`#main` 包進新的 `#body` 容器（保留原本高度撐滿邏輯）。`loadWhoAmI()` 開頁時抓一次帳號名稱與目前狀態渲染按鈕，`toggleReady()` 點擊時 POST 新狀態、用伺服器回傳值（不是樂觀假設）更新按鈕，避免顯示跟伺服器實際值不同步。

**⚠ 實測踩坑：這是使用者真實在用的共用瀏覽器（透過 CDP 接管），不是乾淨的測試環境**——驗證途中同一個 session 的登入帳號在兩次呼叫之間從「系統管理員」變成「HCH」（推測是使用者自己在別的分頁切換了登入身份，因為 JSESSIONID cookie 是整個瀏覽器共用，不是每個分頁獨立）。這不代表功能有 bug（狀態本來就按 `accountId` 分開存，兩個帳號互不干擾），但代表**任何在這個環境裡跑的 Playwright 測試都可能跟使用者的真實操作互相影響**，測試順序、測試中途取得的帳號身份都不能假設是穩定不變的——每次呼叫前如果需要確認是哪個帳號，直接查當次 `/whoami` 回應，不要沿用前一次呼叫記下來的帳號名稱。

### 2026-07-20 改動：「移除」按鈕只給已結束的房間，進行中的房間不能被關掉

使用者反饋「沒有按下『結束服務』的對話不能被我關閉掉（× 不要顯示）」——上一版每個房間項目都有 × 按鈕，容易誤觸把還在進行中的對話從清單藏起來（正是下面「已踩過的坑」那次事故的根因）。改法很單純：`loadRooms()` 組 HTML 時把 `<div class="dismiss">...</div>` 那段包進 `(rm.closed?...: '')` 三元運算，只有 `rm.closed===true` 才輸出；`.room{padding-right:30px}` 這條原本所有房間都會留給 × 按鈕的空間，也一併改成只有 `.room.closed` 才加，避免進行中的房間留白留了個沒用的縫。

### 2026-07-20 改動：未就緒時擋掉「轉真人客服」——不能進線

需求：「在未就緒的情況下面是不能進線的」——如果目前沒有任何客服標記就緒，使用者在 LINE 端打「真人客服」不應該真的轉接（不建立原生 `TcChatRoom`），避免訊息進了真人佇列卻沒人看到。

**判斷「有沒有人就緒」用最簡單的「只要有一個就緒即可」語意**（不是「當下處理這個使用者的那個客服」——這套系統本來就沒有「指派給誰」的概念，主控台是所有客服共用同一份房間清單）：

```java
// LineIntegrationHome 新增
public static boolean isAnyAgentReady() {
    return agentReadyStatus.containsValue(Boolean.TRUE);
}
```

**攔截點在 `LineWebhookServlet.lineAnswer()` 的 `isAgentRequest(lower)` 分支**（判斷使用者說了「真人/客服/轉接/人工」、且目前不在原生房間裡的那個分支），`escalateToAgent()` 之前先擋：

```java
if (isAgentRequest(lower)) {
    insertMessage(roomId, "In", "User", text, "Text", null);
    if (!LineIntegrationHome.isAnyAgentReady()) {
        String msg = LineIntegrationHome.getNoAgentMessage();   // 硬編碼預設訊息，沒有走 TcLineSetting 設定頁
        LineIntegrationHome.pushLine(userId, LineIntegrationHome.textMessage(msg));
        insertMessage(roomId, "Out", "System", msg, "Text", "system");
        return;   // 不呼叫 escalateToAgent()，不建立原生 TcChatRoom，留在 AI 模式
    }
    escalateToAgent(userId, text);
    return;
}
```

**只擋「新升級」這個時間點，不影響已經在真人模式的對話**——`lineAnswer()` 一開始就先查 `nativeRoomId = findOpenNativeChatRoomId(userId)`，如果已經有開啟中的原生房間（代表已經有客服在跟這個人聊），所有後續訊息一律直接轉發進原生房間（`nativeRoomId != null` 那個更早的分支），完全不會走到這個就緒檢查——就緒狀態的變化（客服中途切成未就緒）不會把正在進行的對話腰斬。

**⚠ 重啟後預設是「沒有人就緒」**：`agentReadyStatus` 是純記憶體 Map，`docker restart aipower-app` 之後是空的，`isAnyAgentReady()` 回傳 `false`，所有「轉真人客服」的請求都會先擋下來、直到有客服在主控台按過一次「就緒」為止——這是刻意的安全預設（沒人在看，就不要真的把使用者導去真人佇列），部署後记得提醒客服要記得按就緒。

**驗證方式**：手動組一個帶正確簽章的假 webhook POST（`X-Line-Signature` 用 `line-integration.properties` 裡的 `line.channelSecret` 算 HMAC-SHA256）送「真人客服」給一個全新的假 LINE userId，重啟後（`agentReadyStatus` 必為空）確認：`TcChatRoom`（原生表）完全沒新增一筆，`TcLineChatRoom`（AI 模式記錄表）有這筆對話但 `FMode='ai'`——證實沒有真的轉接。

```bash
SECRET="<line.channelSecret>"
BODY='{"events":[{"type":"message","source":{"userId":"<測試用假ID>"},"message":{"type":"text","text":"真人客服"}}]}'
SIG=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$SECRET" -binary | openssl base64)
curl -s -X POST "http://localhost:22821/aipower/gateway" \
  -H "Content-Type: application/json" -H "X-Line-Signature: $SIG" -d "$BODY"
```

### ⚠ 已踩過的坑：「移除」按鈕全部點掉 = 清單看起來完全空白，沒有任何提示（2026-07-20 真實案例）

**症狀**：使用者回報「用 LINE 傳訊息，聊天框沒有字出來」，一度以為是 webhook 沒收到訊息。查證流程：
1. 直接查 DB（`TcChatRoom`/`TcChatMessage`）——訊息確實有寫入，時間點也對得上。
2. 直接 curl `/console/rooms`、`/console/messages`（本機 + 對外網域都測）——回傳資料完全正確。
3. 這才確認是純前端問題，用 `browser_evaluate` 深入使用者**真實瀏覽器**裡那個 iframe 的 `document.getElementById('rooms').innerHTML`——是空字串。
4. 檢查 `localStorage.getItem('lineConsoleDismissed')`——兩個房間的 ID 都在裡面。**使用者自己（或先前測試時）把僅有的兩個房間都按了「×」隱藏**，房間清單篩選後變成 0 筆，畫面上卻沒有任何文字說明「為什麼是空的」，跟「本來就沒有房間」長得一模一樣，才會被誤認為是資料遺失。

**除錯過程中的一個工具陷阱**：一開始想從外部用 `iframe.contentWindow.BASE`／`iframe.contentWindow.dismissed` 檢查頁面內部狀態，結果 `BASE` 讀出來是 `undefined`——**`const`/`let` 宣告的頂層變數不會掛到 `window` 物件上**，只有 `function` 宣告（如 `loadRooms`）才會變成 `window` 的屬性，這是 JS 本身的行為，不是這支頁面的 bug。要檢查/操控 iframe 內部這類變數，唯一可靠的方式是直接呼叫已掛在 `window` 上的函式（`await win.loadRooms()`）或改 `localStorage` 後**整個 reload 該 iframe**（`iframe.contentWindow.location.reload()`），讓頂層 `let dismissed=loadDismissed()` 重新從乾淨的 `localStorage` 執行一次——不要嘗試從外部直接覆寫 `win.dismissed`，那只會在 `window` 上新增一個沒人用到的同名屬性，完全不影響頁面內部真正在用的那個變數。

**修法**：`loadRooms()` 加一段「全部房間都被 dismissed 篩掉時」的提示——`#rooms` 顯示「N 個房間已隱藏，全部顯示」，點連結呼叫 `showAllDismissed()` 清空 `dismissed` Set 並存回 `localStorage`。這樣以後再發生同樣情況，畫面上至少會告訴使用者「不是沒資料，是被你自己隱藏了」，而不是靜默留白。

### 2026-07-20 版面再調整：padding 縮小 2/3

使用者反饋間距太大，把上一次「加 padding」那次的數值全部除以 3：`#agentBar` 10px 18px→3px 6px；`#body` padding/gap 12px→4px；`.room` 12px 14px→4px 5px；`#msgs` 20px→7px；`.bubble` padding 10px 14px→3px 5px、margin-bottom 12px→4px；`#composer` 14px→5px。純數值調整，佈局結構（卡片式、圓角）不變。

### 已知踩坑：Playwright ref 對這頁面容易失效

`LineChatConsoleServlet` 的頁面有 `setInterval(loadRooms, 3000)`（每 3 秒整個重繪 `#rooms` 的 innerHTML）。用 `browser_snapshot` 拿到 ref 之後，只要點擊動作沒有在 3 秒內完成，`browser_click` 常常會噴 `Ref exx not found`。**對這支頁面做互動測試優先用 `browser_evaluate` 搭配 `document.querySelector(...).click()`**，不要依賴 snapshot ref。

## 這台環境目前的自訂設定（累積記錄，改動時間順序）

- **2026-07-18** `application.properties`（`extension/aipower/config/application.properties`）：
  - `admin.run-sql = true`（原本被註解掉，`product` 模式下預設 `false`）
  - `application.mode = develop`（原本 `product`）——連帶效果：clientData JSON 輸出改成未壓縮格式、執行SQL 不用過 captcha、**Unit 變成可刪除**（product 模式下 Unit 受保護不能刪，develop 模式沒有這層保護，操作上要小心誤刪）。
  - 兩者都改完要 `docker restart aipower-app`，且瀏覽器端要**重新走登入表單登入**（單純 F5 或 `browser_navigate` 不會重建 session/設定快取）。
  - 「執行SQL」被禁用的根本原因（`Configuration.isRunSqlEnabled()` 硬編碼檢查）詳見 `reference_aipower_admin_run_sql_toggle.md` 記憶。
  - 原始檔案備份在同目錄 `application.properties.bak`。
- **2026-07-18** `TsMenu` 新增了一個頂層頁籤「開發」（`FId=b71be455-838d-48ca-8439-f90e941e4cf7`，`FTreeLevel=1`，`FTreeSerial=004`）+ 底下一個 Level-2 子目錄「工具」（`FId=4161ccaa-8d2b-426e-b8ec-425d5fc5deb3`，`FTreeSerial=004.001`）。建立頂層目錄時踩到「Directory 節點沒有子節點就完全不會顯示在側邊欄」這個坑（不管 `FEnabled=1` 還是重整頁面都沒用，必須先建至少一個子節點），完整操作手法與這個坑的細節見 `ecp-menu` skill 的「Creating a Brand-New Top-Level Group」章節。
- **2026-07-19** 新增兩個標準六大核心 Unit，Service 層直接呼叫 MikoPBX（本機另一個 Docker 專案，`mikopbx2026` 容器）的 REST API 同步資料，掛在「開發→工具」底下：
  - `Ecp.MikopbxExtension`（`FId=18cffa48-ecb1-4aba-b53e-2ba5738d03b8`，實體表 `TcMikopbxExtension`）：分機清單 + 即時 SIP 狀態，一列一支分機，每次同步覆蓋更新。
  - `Ecp.MikopbxCallLog`（`FId=67747b8f-e602-439d-bd1f-142a7d8897e8`，實體表 `TcMikopbxCallLog`）：通話紀錄，用 `UNIQUEID` 去重只新增不覆蓋。
  - 兩者共用同一份 `extension/aipower/config/mikopbx-integration.properties`（`mikopbx.api.baseUrl=https://mikopbx-2026-docker:18443`、`mikopbx.api.token=<Bearer token>`）。
  - **✅ 已修**：容器內 `/etc/hosts` 那條 `192.168.65.254 mikopbx-2026-docker` 已經加進 `D:\Dockers\aipower\docker-compose.yml` 的 `app` service `extra_hosts:`，`docker compose up -d` 套用過、實測確認容器重建後這條記錄自動存在，不用再手動補。**如果之後又看到清單空空的**，先查 `docker logs aipower-app | grep -i mikopbx` 有沒有 `SSLHandshakeException`，代表 extra_hosts 設定不知為何又失效了。
  - **2026-07-19 追加第三個 Unit `Ecp.MikopbxSetting`**（`FId=19ecbd71-17d1-4bd0-a583-bfdcf8eb5e3b`，實體表 `TcMikopbxSetting`）：MikoPBX 系統設定，**唯一的「寫」方向 Unit**——單筆固定記錄（`UUID.nameUUIDFromBytes("mikopbx-general-settings")` 當固定 FId），開清單頁 GET 同步，Form 頁按保存會 PATCH 回 MikoPBX。可編輯：Name/Description/PBXLanguage/PBXAllowGuestCalls/PBXRecordCalls/PBXRecordCallsInner/PBXInternalExtensionLength/SIPPort/TLS_PORT/RTPPortFrom/RTPPortTo/SIPDefaultExpiry；唯讀顯示：WebAdminLogin；完全不放：密碼/金鑰/憑證/AMI/ARI/IAX。寫回設計（override `doUpdate()` 而非 `update()`）、PATCH 語意查證、SIPPort 存檔數秒內自動生效（不需重啟）的實測發現，完整寫在上一節「建一個標準六大核心 Unit」第 6 節。
  - 完整建置手法（六大核心怎麼寫、為什麼放棄自訂按鈕改用 `prepareList` 自動同步、HTTPS 自簽憑證兩層坑、寫回外部系統的 `doUpdate()` override 模式、部署流程）見上一節「建一個標準六大核心 Unit，Service 層呼叫外部 HTTPS API」。Java 原始碼/SQL/properties 範本存在 `C:\Users\HCH\.claude\skills\aipower-docker-local\mikopbx-integration\`（`src\`、三份 `mikopbx_*_unit.sql`、`mikopbx-integration.properties`），照抄改個 UUID/package 名稱就能建下一個「呼叫外部 API 讀寫」的 Unit。
- **2026-07-20** `LineChatConsoleServlet`（`/aipower/line/console`，見上面「LINE 客服主控台」章節）房間清單加「移除」按鈕：純前端 `localStorage` 隱藏已看過的房間，不動 `TcChatRoom` 資料庫欄位。編譯部署後 `docker restart aipower-app`，Playwright 實測確認：點 `.room .dismiss` 後該房間從清單消失、重新整理頁面仍維持隱藏（`localStorage` 持久化），且不影響其他瀏覽器/其他客服看到的清單。
- **2026-07-20** 同一支 servlet 加「目前登入帳號」顯示 + 「就緒/未就緒」切換鈕（見上面「抓目前登入帳號 + 就緒/未就緒狀態」章節）：`LineIntegrationHome.getCurrentOnlineUser()` 用官方 `OnlineUserHome.getService().getItem(session)` API 取帳號，狀態存 `LineIntegrationHome` 內一個按 accountId 分的記憶體 Map（重啟清空，故意不進 DB）。新增 `GET .../whoami`、`POST .../ready` 兩個路由。Playwright 實測透過 `https://hch.james-huang.org` 正式網域登入（不是 `127.0.0.1:22821` 直連，兩者 cookie 網域不同、session 不共通）驗證通過：切換後狀態即時反映在按鈕顏色，且是伺服器端持久（重新整理/直接呼叫 API 都拿得到同一個值），不是純前端假象。
- **2026-07-20** 版面微調（使用者反饋「要有 padding」「title 顏色太深」）：`#agentBar` 背景從深色 `#2d3a4a` 改成淺灰藍 `#7c8ba3`；`#body` 加 `padding:12px;gap:12px`，`#rooms`/`#main` 從貼齊邊緣的平面佈局改成卡片式（各自 `border-radius:8px` + `border`），`#main` 補上白底（原本沒設背景，訊息區會透出頁面底色）；`.bubble`/`#msgs`/`#composer` 的 padding 也一併加大。純 CSS 調整，沒動任何 JS 邏輯或後端路由。
- **2026-07-20** 除錯真實案例＋修正：使用者回報「LINE 傳訊息後聊天框沒字」，查證發現訊息其實正常寫入 DB、API 也正常回傳，根因是**兩個房間都被前一版新增的「移除」按鈕隱藏掉了**，`localStorage` 清單清空後房間又消失得無聲無息（見上面「已踩過的坑」章節，含 `const`/`let` 頂層變數不會掛到 `window`、只能改 `localStorage` 後整個 reload iframe 才能重置這個工具陷阱）。修正：`loadRooms()` 加「N 個房間已隱藏，全部顯示」提示＋一鍵清空 `dismissed`，避免以後同樣情境又被誤認為資料遺失。同一批改動也把上一版加大的 padding 依使用者要求整體縮小為原本的 1/3（見上一節「版面再調整」）。
- **2026-07-20** 承上一次事故，「移除」× 按鈕改成只在 `rm.closed===true` 才輸出（進行中的房間完全不給關閉的機會），從源頭杜絕再次誤關掉還在服務中的對話（見上面「『移除』按鈕只給已結束的房間」章節）。
- **2026-07-20** ✅ **LINE SOCKS proxy 已整組移除，改為直連——Fortinet 封鎖已解除，這是目前的正確架構**。經過：使用者要求關掉 8888 的 proxy，查到 `ssh.exe -D 8888` 連的是 **`hch@10.145.119.12`（IP 打錯，正確是 `.19`）**，且 process 已自己死掉；依指示一併刪掉 `netsh` portproxy 的 `0.0.0.0:8889 → 127.0.0.1:8888` 規則（需 UAC 提權）。要接回去時發現 **`.19` 主機關機中**（本機 ZeroTier 正常、同網段手機 `.96` ping 得到 45–149ms，唯獨 `.19` 100% 遺失＋SSH 逾時 → 遠端離線，非本機問題）。使用者接著要求「不要透過 proxy 上網，把所有透過他的都刪了」，**刪除前先實測封鎖是否還在，結果發現已解除**：host `curl https://api.line.me/...` → HTTP 401、容器內 `openssl s_client` + HEAD → HTTP/1.1 401、host 與容器 DNS 都解析到真實 Akamai IP `23.210.238.39`（不是當初的 sinkhole）。於是把 `LineIntegrationHome.java` 的 `LINE_API_PROXY` 欄位刪除、`rawHttpsCall()` 從 `new Socket(LINE_API_PROXY)` + `createUnresolved()` 改回 `new Socket()` + 一般 `InetSocketAddress`（只有這兩行是 live 的 proxy 程式碼，`import java.net.*` 是萬用字元故不受影響）。編譯部署 `LineIntegrationHome.class` + `LineIntegrationHome$1.class`（匿名 TrustManager 內部類，**兩個都要 cp，漏了會 NoClassDefFoundError**）後 restart，**用真實 channel token 唯讀呼叫 `GET /v2/bot/info` 驗證：HTTP 200，回傳 `{"basicId":"@593pysxw","displayName":"AI3.5",...}`**，全鏈路直連確認可用。**附帶好處**：拿掉了一個單點故障——ssh 通道不是常駐服務，主機關機/斷線就死，一死 `pushLine()` 就丟 `Malformed reply from SOCKS server`、連帶讓 `handleClose()` 炸掉、房間永遠關不掉（見上面「結束服務」那則的根因 2）。
- **2026-07-20** ✅ **移除 LineIntegrationHome 的 trust-all SSL，恢復正常憑證驗證**。原註解宣稱「容器 JRE 預設 cacerts 驗證 `api.line.me` 憑證鏈失敗（PKIX path building failed），但 `api.deepseek.com` 沒問題」，據此用了 `checkServerTrusted` 空實作的 trust-all `SSLContext`。**這個診斷是錯的**：真正原因是當時 **Fortinet 正在攔截 `api.line.me`，回的是它自己簽的憑證**，Java 當然驗不過（DeepSeek 不在封鎖名單，所以只有 LINE 出事——這個「只有一個網域壞掉」的模式本身就是攔截的指紋，不是 truststore 壞掉的樣子）。封鎖解除後實測預設 truststore 完全正常：`TLSv1.3`、`CN=api.line.me, O=LY Corporation`、`issuer=DigiCert Global G3 TLS ECC SHA384 2020 CA1`、2027-01-07 到期。改成 `SSLSocketFactory.getDefault()` 並移除 5 個變成未使用的 import（`SSLContext`/`TrustManager`/`X509TrustManager`/`SecureRandom`/`X509Certificate`）。**注意編譯產物變了**：匿名 `X509TrustManager` 內部類消失，`LineIntegrationHome$1.class` 不再產生，容器內那個舊的要手動 `rm`（否則變成新的殘留）。**驗證手法（值得重用）**：寫一支 `LiveProbe` 用**反射**呼叫容器內「已部署」的 private `rawHttpsCall()` 打唯讀端點 `/v2/bot/info`，這樣測到的是真正的程式碼路徑，不是 `curl`/`openssl` 的網路層——結果 `RESULT=OK` 並回傳完整 bot 資訊。**教訓：看到 `PKIX path building failed` 先查 issuer 是誰（是不是中間設備），不要反射性地改用 trust-all。**
- **2026-07-20** 全機健檢＋清理（使用者要求「清掉多餘的、修掉有問題的」）。**清掉的**：孤兒 metadata 共 19 筆（`tsfield` 2、`tsimporttemplate` 13、`tskeywordfield` 4、`tsslaveunitprivilege` 1，來自 5 個 ghost unit）；容器內 `.class.bak-*` 與 `application.properties.bak`；7 個已結束的測試聊天室 + 42 則訊息；容器 `/tmp` 積存的診斷檔。全部先備份到 `D:\Docker BAK\aipower-cleanup-20260720\`。**⚠️ 兩個「看起來像孤兒但絕對不能刪」的**：(1) `FUnitId='00000000-0000-0000-0000-000000000000'` 出現在 `tshomepageitem`/`tshomepageformat`/`tshomepageitemarrangement`，是**首頁版面配置的哨兵值**（代表「不隸屬任何 Unit」），刪掉會弄壞首頁；(2) `277f1cb5-bcf6-47cf-a7dd-de528ee33937` 出現在 `tcproduct`，是**業務資料**（2 筆真實產品），sweep 腳本本來就拒碰 `tc*/tp*/tt*`。**方法論教訓**：ghost unit ID 清單**不能只從 `TsField` 推導**——第一次只掃 TsField 得到 2 個 ID，掃完才發現 `TsImportTemplate`/`TsKeywordField` 還引用著另外 3 個完全不同的 ghost unit。正確作法是對「所有含 `FUnitId` 欄位的表」做 UNION 全表列舉，再逐一判斷哪些是真孤兒、哪些是哨兵/業務資料。**確認健康、不用動的**：Tomcat 只有 `aipower`+`ROOT`（無 docs/examples/manager）、TsMenu 零斷鏈、無孤兒聊天訊息、MikoPBX 整合 `https://mikopbx-2026-docker:18443` 回 200、log 零錯誤、無 dangling image/停止容器。`alpine:latest`（13MB，唯一未使用的 image）**刻意保留**——Docker 備份/還原流程要用它掛 volume 打包，刪掉只省 13MB 卻要重新 pull。
- **2026-07-20** 順帶盤點過本機 proxy 設定，確認**系統層級沒有任何 proxy 生效**：環境變數無 `HTTP_PROXY`/`HTTPS_PROXY`；WinINET `ProxyEnable=0`（`ProxyServer` 欄位殘留公司 proxy `10.145.119.209:808` 但未啟用）；`netsh winhttp show proxy` 為 Direct access。當時全機唯一在走 proxy 的就只有上面那條 LINE API 的 SOCKS，已移除。
- **2026-07-20** 使用者回報「對話框太長了，用一半的 size」。**根因是 CSS 選擇器沒有限定範圍**：`.Outer{background:#fff;border:...}` / `.Inner{background:#95ea69}` 這兩條沒有前綴，而 `loadMessages()` 同時把 `m.category`（`Outer`/`Inner`）掛到 `row.className='msgRow '+m.category` **和** `b.className='bubble '+m.category` 上，所以整個 `.msgRow`（`display:flex`，寬度撐滿）也吃到了背景色與邊框——看起來就是每則訊息都是一個滿版色塊，`.bubbleWrap` 的 `max-width` 完全被無視。修正：兩條改成 `.bubble.Outer{...}` / `.bubble.Inner{...}`，並把 `.bubbleWrap` 的 `max-width` 從 `min(65%,420px)` 收成 `min(50%,320px)`。**教訓**：這頁的 `Outer`/`Inner` 是同時貼在 row 與 bubble 上的共用 class 名，任何針對它們寫的樣式一律要加 `.bubble` 或 `.msgRow` 前綴限定，否則會同時命中兩層。
- **2026-07-20** 就緒/未就緒狀態第一次真正被消費：`LineWebhookServlet.lineAnswer()` 的「轉真人客服」升級點加 `LineIntegrationHome.isAnyAgentReady()` 守門，沒有任何客服就緒就不建立原生 `TcChatRoom`、回一則「客服都不在線上」訊息、留在 AI 模式（見上面「未就緒時擋掉『轉真人客服』」章節）。用手動組簽章的假 webhook POST 實測驗證：`TcChatRoom` 完全沒新增、`TcLineChatRoom` 停在 `FMode='ai'`。⚠ 重啟後 `agentReadyStatus` 是空的，等同「預設沒人就緒」，部署後要提醒客服記得按就緒鈕。
- **2026-07-20** ⚠ **根因修正**：使用者實際在主框架選單裡點開「LINE 聊天框」分頁（`TsMenu.FExternalPageUrl` 開的 iframe），畫面一直顯示「未登入」、就緒鈕按了完全沒反應，即使外層 ECP 主框架右上角明明顯示已登入帳號名稱。查證發現 `TsMenu` 那筆選單（`FId=19f7e89f-e3f0-07f7-2e00-863e3a3759d8`）的 `FExternalPageUrl` 被寫死成絕對網址 `https://hch.james-huang.org/aipower/line/console`，跟主框架實際載入的來源 `http://127.0.0.1:22821` 是不同 origin，瀏覽器完全不會把 `127.0.0.1:22821` 的 session cookie 帶進這個 iframe，所以 `/whoami`、`/ready` 永遠收到匿名請求、回 401。**修法**：把 `FExternalPageUrl` 改成同源相對路徑 `/aipower/line/console`（不寫死協定+網域），iframe 就會自動跟著外層當下的來源走，不管是走 `127.0.0.1:22821` 直連還是走 Cloudflare Tunnel 網域都能正確帶到當下那個 session 的 cookie：
  ```bash
  MSYS_NO_PATHCONV=1 docker exec aipower-mariadb mariadb -uroot -p<DB_PASSWORD> default -e \
    "UPDATE tsmenu SET FExternalPageUrl='/aipower/line/console' WHERE FId='19f7e89f-e3f0-07f7-2e00-863e3a3759d8';"
  ```
  這是純 DB 設定改動，不用重啟、不用重新編譯，改完後**要整個關掉「LINE 聊天框」分頁再重新點選單打開**（或用 `document.querySelector('iframe[src*="line/console"]').src='/aipower/line/console'` 強制該 iframe 重新載入），Playwright 實測確認：改完後主控台立刻抓到「系統管理員」帳號、就緒鈕點擊後能正常切換成「就緒」。**教訓**：任何用 `FExternalPageUrl`／iframe 嵌入、且該頁面需要讀取當前登入 session 的自訂頁面，網址一律用相對路徑（同源），絕對不要寫死成 Cloudflare Tunnel 這類外部網域——那只適合「獨立分頁直接開」的場景（例如純測試用瀏覽器打開 `https://hch.james-huang.org/...`），一旦被嵌進主框架當 iframe 用，寫死外部網域就必然導致 session 對不上。

- **2026-07-20** ⚠ **「結束服務」按了沒反應，真正原因是兩個根因疊加，缺一個都修不好**：
  1. **前端**：`closeRoom()` 原本用瀏覽器原生 `window.confirm('確定結束此服務？')`。**只要這個分頁當下有 Playwright/CDP（remote debugging）連著在操作**，Chrome 會把後續同分頁裡真人觸發的 `confirm()`/`beforeunload` 對話框處理成「不對真人顯示、只掛在 CDP 的待處理佇列」——真人完全看不到彈窗，JS 執行卡住，點幾次「結束服務」就疊幾個看不到的對話框在佇列裡，永遠不會真的呼叫到 `/close`。**修法**：改成頁面自己畫的確認方塊（`#confirmOverlay`/`#confirmBox` + `customConfirm(msg)` 回傳 Promise），不再依賴瀏覽器原生 dialog，不管有沒有人在用 debug Chrome 連著都不受影響。
  2. **後端**：即使前端彈窗修好、真的送出 `/close` 請求，`handleClose()` 會先呼叫 `LineIntegrationHome.pushLine()` 推播「已結束真人客服」訊息給使用者，**成功了才會執行 `closeNativeChatRoom(roomId)`**。這台 Docker host 所在網路的 Fortinet Secure DNS 會擋 `api.line.me`，所以 LINE API 呼叫是走 Windows host 上手動開的 `ssh -D 8888 hch@10.145.119.19` SOCKS5 動態代理，再用 `netsh interface portproxy`（`0.0.0.0:8889 → 127.0.0.1:8888`）轉發給容器透過 `host.docker.internal:8889` 存取（程式碼在 `LineIntegrationHome.java` 第 92-100 行有完整註解）。**這條 `ssh.exe` 通道不是常駐服務，關機/斷線/session 結束都會死掉**，一旦死掉，`netsh portproxy` 規則還在但轉發目標（127.0.0.1:8888）沒人聽，容器那邊的 Java SOCKS client 收到空/畸形回應，丟出 `LINE API 呼叫失敗: ... Malformed reply from SOCKS server`，`handleClose()` 在 `pushLine()` 這步就整個炸掉、`closeNativeChatRoom()` 永遠不會執行到——**房間在 DB 裡會一直卡在「開著」，不管前端重試幾次都一樣**，症狀跟「彈窗看不到」幾乎一模一樣，容易搞混成同一個問題，其實是兩件事。**排查方法**：`docker logs aipower-app`（或容器內 `/app/apache-tomcat/logs/catalina.<日期>.log`）搜尋 `LineChatConsoleServlet error on /line/console/close`，看到 `Malformed reply from SOCKS server` 就是這個。**修法**：在 Windows host 重開通道（背景常駐）：
     ```bash
     nohup ssh -N -D 127.0.0.1:8888 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes hch@10.145.119.19 > /tmp/ssh-socks-tunnel.log 2>&1 &
     disown
     ```
     驗證通道打通（透過 8889 portproxy，不用進容器）：
     ```bash
     curl -s --socks5-hostname 127.0.0.1:8889 -o /dev/null -w "HTTP %{http_code}\n" https://api.line.me/v2/bot/info -H "Authorization: Bearer dummy"
     # 只要回 401（不是 timeout/connection refused）就代表 SOCKS 鏈路正常，401 是 LINE API 嫌 token 是假的，不是網路問題
     ```
     `netstat -ano | grep :8888` 沒有 LISTENING 就代表通道死了；`netsh interface portproxy show all` 可以確認 8889→8888 這條轉發規則還在（這條規則本身很少掉，通常是上游的 `ssh.exe` process 死掉）。

     ⚠️ **以上整段（根因 2 的 SOCKS proxy 部分）自 2026-07-20 下午起已不適用**：Fortinet 對 `api.line.me` 的封鎖已解除，實測直連可回 HTTP 200，程式碼裡的 SOCKS proxy 已移除、`ssh` 通道與 `netsh portproxy` 規則也都拆掉了（見下方同日 ✅ 條目）。**現在如果「結束服務」又沒反應，不要再往 SOCKS 方向查**——先看 `docker logs aipower-app` 的實際例外訊息。

     只有在 Fortinet 封鎖**重新出現**時（症狀：連線逾時，或收到 Fortinet 的 "Web Page Blocked" HTML）才需要恢復這套機制，且要三件事一起做：(1) `.19` 開機並上 ZeroTier、(2) 跑上面那行 `ssh`、(3) 用**系統管理員權限**補回轉發規則（一般 shell 會回 `The requested operation requires elevation`，可用 `Start-Process netsh -Verb RunAs -Wait -ArgumentList ...` 觸發 UAC），並把 `rawHttpsCall()` 改回 proxy 版本：
     ```
     netsh interface portproxy add v4tov4 listenport=8889 listenaddress=0.0.0.0 connectport=8888 connectaddress=127.0.0.1
     ```
  - 兩個修法都不用 `docker restart` 就能生效前端部分（`ssh` 通道是 host 端的事，跟容器重啟無關；但這次前端 `.class` 有改，還是照常編譯部署 + `docker restart aipower-app`）。Playwright 實測：先用直接呼叫 `win.customConfirm()`／`win.closeRoom()` 確認彈窗會正常顯示、SOCKS 通道修好後完整跑一次真實關閉流程，查 DB `TcChatRoom.FCloseTime` 確認真的寫入時間戳，全鏈路驗證通過。

- **2026-07-20** 房間清單可收合 + 聊天氣泡改成真實 LINE 樣式：`#rooms`/`#main` 之間加一個 `#roomsToggle`（‹/›）小按鈕，點擊切換 `#rooms.collapsed`（`width:0;opacity:0`）並存 `localStorage['lineConsoleRoomsCollapsed']` 持久化。氣泡樣式來回改了三版才定案（使用者陸續給了截圖回饋）：第一版加 🙂/🎧 表情符號頭像＋固定 60% max-width，因為容器變寬導致氣泡撐到接近滿版（`max-width:60%` 是相對容器寬度算的，不是絕對值）被嫌醜；第二版改用漫畫尖尾巴+膠囊形（`clip-path` 三角形 tail），使用者嫌尾巴/膠囊太長；**最終版**改成貼近真實 LINE App 的樣式（使用者直接截自己手機 LINE 對話當範本）：一般圓角矩形（`border-radius:16px`，靠對方那一側角落縮成 `4px` 模擬指向感，不用額外的三角形 tail 圖形）、`.bubble{width:fit-content;max-width:100%}` 包在 `.bubbleWrap{max-width:min(65%,420px)}` 裡讓氣泡貼合文字內容而不是撐滿、客戶端（`Outer`）訊息左側加一個 30px 圓形頭像（沒有真的大頭貼可用，用聯絡人名稱首字當替代——`LineIntegrationHome`/DB 都沒存 LINE 個人頭像網址，只有 `fetchProfileName` 抓顯示名稱，要顯示真大頭貼得額外接 LINE Profile API 存 `pictureUrl`，目前沒做）、客服端（`Inner`）不放頭像靠右對齊，每則訊息下面加 `fmtTime()` 從 `FSendTime` 切出的 `HH:mm` 時間戳。**教訓**：改 UI 風格這種主觀設計類需求，与其自己猜測直接改完部署，不如先跟使用者要一張截圖/參考圖（或至少問清楚是形狀、顏色、尺寸、還是位置的問題）再動手，能省下好幾輪來回編譯部署。

- **2026-07-21** ✅ **新增「進線 ACD」——LINE 真人客服從「共用清單、客服自己挑房間」改成
  round-robin 自動指派**。使用者要求「用六大原則的方式寫一個進線的acd」，因此新建了兩個標準
  六大核心 Unit（`com.chainsea.ecp.acd.agent`／`com.chainsea.ecp.acd.assignment`，package 內
  各含 Model/Dao/Service/Action/Home）：
  - **`Ecp.AcdAgent`**（表 `TcAcdAgent`，`FId=8d653e19-e8fa-44ee-b09f-7d934a2b6737`）：每位客服
    一列，`FStatus`（Ready/NotReady）＋`FLastAssignTime`（round-robin 排序用，NULL 或最舊者優先）。
  - **`Ecp.AcdAssignment`**（表 `TcAcdAssignment`，`FId=a0d5983c-8ab8-40e1-a4f8-04153aeae7df`）：
    每通進線一列，`FStatus`（Waiting/Assigned/Closed）記錄目前排隊中還是已指派給誰。
  - **引擎邏輯刻意不放在 Service 層，放在兩個 Home 的 static 方法**（`AcdAgentHome.setReady/
    setNotReady/isReady/isAnyAgentReady`、`AcdAssignmentHome.enqueueAndDispatch/dispatch/
    closeAssignment/getRoomIdsForAgent`），直接用 `LineIntegrationHome.getDbConnection()` 做
    JDBC（`SELECT...FOR UPDATE`+交易避免多人搶同一位客服）。原因：唯一呼叫端
    `LineWebhookServlet`/`LineChatConsoleServlet` 都是原生 HttpServlet，前者完全匿名、後者雖有
    ECP 登入 session 但同樣沒有框架自動注入的 ServiceContext——跟 `LineIntegrationHome` 開頭
    註解說明的既有理由完全一致，是同一慣例的延伸，不是破例。Service/Dao/Action 三層仍完整存在
    且可正常繼承標準 CRUD（供未來若要開管理頁查詢 `TcAcdAssignment` 歷史記錄時使用），只是目前
    没有規劃 List/Form/Menu/Privilege（無獨立管理頁面需求，YAGNI）。
  - **`LineIntegrationHome` 的舊版純記憶體 `agentReadyStatus`（2026-07-20 那版）已整個移除**，
    改由 `TcAcdAgent` 接手（重啟不再清空，是本次附帶的正向改變）。`NO_AGENT_MESSAGE`／
    `getNoAgentMessage()` 重新命名為 `AGENT_QUEUED_MESSAGE`／`getQueuedMessage()`，語意從「沒人
    在線」改成「已為您排隊」。
  - **行為變更（刻意）**：`LineWebhookServlet.lineAnswer()` 原本「沒人就緒就整個拒絕轉真人」的
    硬性擋門已拿掉，改成一律建立原生房間＋呼叫 `AcdAssignmentHome.enqueueAndDispatch()`——有就緒
    客服就立即輪流指派，沒有就留在 Waiting 佇列，等任一客服下次按「就緒」時自動補派
    （`setReady` 內部會呼叫 `dispatch()`）。這才是使用者要的「客服就緒在線、然後等待分配」。
  - **`LineChatConsoleServlet` 房間清單改成只顯示指派給目前登入客服的房間**
    （`AcdAssignmentHome.getRoomIdsForAgent`，`/console/rooms` 加了 401 未登入檢查），還在
    Waiting 佇列、尚未指派的房間任何客服都看不到——避免還沒輪到的人搶先接走，這正是 ACD 的意義。
    `/console/close` 額外呼叫 `AcdAssignmentHome.closeAssignment()` 收尾。
  - **實測驗證**（用假 LINE userId + 正確簽章模擬 webhook，經 Playwright 用真實登入 session 呼叫
    `/whoami`/`/ready`/`/rooms`/`/close`，並直接查 DB 交叉核對）：(1) 沒有客服就緒時新進線正確
    落在 `TcAcdAssignment.FStatus='Waiting'`、`FAgentAccountId` 為 NULL；(2) 客服按「就緒」後，
    佇列裡最舊的一筆立即被指派給該客服、`TcAcdAgent.FLastAssignTime` 同步更新；(3) 客服已就緒時
    新進線立即指派（不進 Waiting）；(4) 房間清單正確只顯示指派給當前登入客服的房間。`docker logs`
    確認 `Register unit: Ecp.AcdAgent`／`Ecp.AcdAssignment` 皆成功、過程中無新增例外（既有的
    `ChatRoomHome.getService() is null` 等 NPE 是 2026-07-18 就存在、與 aipower.crm 模組未安裝有
    關的已知背景雜訊，與本次改動無關）。**唯一沒能完整測到的是 `/console/close` 成功路徑本身**
    ——測試用的是假 LINE userId，LINE Push API 對不存在的帳號一律回 400，導致 `handleClose()`
    卡在既有的「pushLine 沒成功就不落地」保護邏輯（這是 2026-07-20 就有的既有行為，不是本次改動
    造成），沒有真實 LINE 帳號無法把這條路徑走到底；`AcdAssignmentHome.closeAssignment()` 本身只
    是一條單純的 UPDATE，經程式碼審視風險低，但仍值得記錄「尚未在真實關閉流程下實測」這個限制。
  - 原始碼與新增 SQL：三支既有檔案（`LineIntegrationHome.java`/`LineWebhookServlet.java`/
    `LineChatConsoleServlet.java`）已更新回這個 skill 的 `line-integration/src/` 目錄；ACD 兩個
    Unit 的六大核心原始碼與建表/建 Unit SQL 目前只存在本次 session 的 scratchpad（尚未固化進
    某個 skill 目錄）——之後若要修改 ACD 邏輯，建議先把 `com/chainsea/ecp/acd/` 整包原始碼與
    `acd_unit.sql` 一併存進本 skill 目錄（比照 `mikopbx-integration/` 的做法），避免下次改動要
    重新反推。

- **2026-07-21（續）** 使用者問「ACD 有沒有介面可以設定/調整」，回答「目前沒有，只有客服
  自己切就緒」後，使用者要求加「查看 + 主管可以手動調整」等級的管理介面。新增
  **`AcdAdminServlet`**（`com.chainsea.ecp.acd` 頂層套件，跟 `LineChatConsoleServlet` 同一套
  raw HttpServlet 慣例，理由相同：這個框架版本的 `EntityList.jsp` 沒有 per-unit 自訂按鈕載入
  機制，標準 List/Form 樣板做不到「下拉選客服＋按鈕改派」這種操作），掛在
  `/aipower/acd/admin`（`/aipower/acd/admin/*`），透過既有的 `LineIntegrationStartupListener`
  一併註冊（`setLoadOnStartup(5)`，是該 listener 目前掛的第 3 支 servlet）：
  - `GET /admin` → 管理台頁面（兩張表：客服就緒狀態、最近 100 筆進線指派紀錄，5 秒輪詢）。
  - `GET /admin/agents`／`GET /admin/assignments` → JOIN `TsAccount`／`TcChatRoom` 補上顯示
    名稱，查詢邏輯直接寫在 servlet（跟 `LineChatConsoleServlet.handleRooms` 既有寫法一致，
    不是每個 JOIN 查詢都要包一層 Home 方法）。
  - `POST /admin/agent/ready` → 主管可以切**任何人**的就緒/未就緒（呼叫既有
    `AcdAgentHome.setReady/setNotReady`，這兩個方法本來就吃任意 accountId，不用改）。
  - `POST /admin/assignment/reassign` → 新增 `AcdAssignmentHome.reassign(chatRoomId,
    newAgentAccountId)`：把某房間目前有效（非 Closed）的指派紀錄改派給指定客服，不論原本是
    Waiting 還是已指派給別人，且不檢查目標客服是否就緒（主管可以刻意指派給未就緒的人）。低頻
    人工介入操作，不像自動派工熱路徑的 `tryAssignOne()` 需要 `FOR UPDATE` 鎖。
  - 沒有額外角色權限檢查，跟 `LineChatConsoleServlet` 目前的安全模型一致（任何登入帳號都能開、
    都能調整任何人）——刻意不新增角色概念，YAGNI。
  - **選單**：`TsMenu` 新增「ACD 管理台」（`FId=c6e8ccd1-b345-4c27-a800-3a350a09f0ea`），
    跟「LINE 聊天框」同一層（`辦公自動化` 模組下），`FExternalPageUrl='/aipower/acd/admin'`
    ——沿用 2026-07-20 那次踩過的坑：**一律用同源相對路徑**，不要寫死 Cloudflare Tunnel
    網域，否則 iframe 跟外層來源不同、session cookie 帶不進去（見上面「LINE 聊天框」分頁登入
    session 那次事故）。`TsRoleMenu` 直接複製「LINE 聊天框」那筆的角色權限。
  - **驗證**：`docker logs` 確認 `Register unit: Ecp.AcdAgent`／`Ecp.AcdAssignment` 都成功；
    `curl` 直接測三個端點狀態碼——`/acd/admin`（頁面）200、`/acd/admin/agents`（API，未帶
    session）401，符合預期的登入檢查。`reassign()` 的 SQL 邏輯另外用直接下 SQL 模擬同一句
    `UPDATE ... WHERE FChatRoomId=? AND FStatus<>'Closed'` 驗證過（建一筆 Waiting 測試資料→
    執行→確認變成 Assigned 且 FAgentAccountId 正確→清除測試資料）。**這次收尾時 Playwright
    接管的 Chrome（CDP，`playwright-vrs`）卡住沒回應**（`browser_tabs`/`browser_navigate` 都
    逾時），沒能在瀏覽器裡實際點過新選單項目、看過管理台頁面渲染結果、或走過需要登入 session
    的兩個 POST 端點（就緒切換／改派）的真實前端流程——這幾項**只驗證到後端邏輯與 HTTP 層**，
    UI 呈現與登入態下的完整互動流程之後應該找時間用 `/connect-chrome` 重新接管後補測。

- **2026-07-21（續2）** 兩個使用者回報的即時性問題，都修完並用 Playwright 在真實瀏覽器
  （兩個分頁同時開：LINE 聊天框 + ACD 管理台）驗證通過：
  1. **「聊天框設定就緒後 ACD 管理台沒馬上反映」的反方向**——`LineChatConsoleServlet` 的
     `PAGE_HTML` 原本 `setInterval` 只輪詢房間清單/訊息，就緒狀態只在頁面剛載入時抓一次。
     改成把 `loadWhoAmI()` 一起排進同一個 3 秒輪詢，這樣不管是自己按按鈕、還是被 ACD 管理台
     改掉，畫面都會在幾秒內自動同步（`loadWhoAmI()`/`toggleReady()` 本來就是「以伺服器回應
     為準」寫回畫面，不是樂觀更新，兩者共用同一輪詢不會互相打架）。
  2. **「明明多人上線，ACD 管理台客服清單卻只出現一個人」**——根因：`/admin/agents` 原本
     直接 `SELECT ... FROM TcAcdAgent`，這張表只有**按過就緒/未就緒按鈕**的帳號才有列，
     從沒碰過那顆鈕的在線帳號完全不會出現。改法：`AcdAdminServlet.handleAgents()` 先查 ECP
     原生 `TsOnlineUser`（見 `ecp-schema` skill 的「Employee/Account 元資料」章節）列出**目前
     所有在線帳號**當底（狀態預設 NotReady），再疊上 `TcAcdAgent` 的真實狀態；曾經就緒過但
     現在已經登出的帳號也保留在清單裡（`online:false`），方便主管抓到「顯示就緒但人已經不在
     線上」這種異常。前端加了綠/灰小圓點標示在線/離線。
  - 兩處改動都只動了既有檔案（`LineChatConsoleServlet.java`／`AcdAdminServlet.java`），已同步
    寫回 `line-integration/src/`／`acd-integration/src/`。編譯部署 `docker cp` 單一 class＋
    `docker restart aipower-app`（重啟前都確認過沒有進行中的真人客服對話）。

- **2026-07-21（續3）** 使用者要求「顯示就緒人員的進線順序號」，接著又要求「沒有在線的人
  不能被設定就緒」，兩個都加完並用 Playwright 在真實登入 session 下端到端驗證：
  1. **進線順序號**：`AcdAdminServlet.handleAgents()` 把就緒中的客服依照
     `AcdAgentHome.lockNextReadyAgent()` **完全相同的排序準則**（`FLastAssignTime` 越舊、
     含從沒被指派過者排最前）算出 `queueOrder`（1=下一位），非就緒者 `queueOrder=0`。
     刻意不另外發明一套排序邏輯——畫面數字必須跟系統實際會怎麼派工一致，用假訊息模擬一次
     真實進線交叉驗證過（順序①的客服確實拿到那通對話）。`lastAssignTime` 是
     `"yyyy-MM-dd HH:mm:ss.f"` 字串，字典序排序恰好等同時間排序，空字串（NULL/從未指派過）
     字典序上自然排最前，不用另外解析成 Date。前端用綠色圓形數字徽章顯示。
  2. **離線者不能設就緒**：`handleSetAgentReady()` 新增 `isAccountOnline()` 檢查
     （查 `TsOnlineUser`），`ready=true` 且該帳號不在 `TsOnlineUser` 時回 409。**設為
     未就緒不受此限制**——離線但殘留 `Ready` 狀態的帳號（例如忘記按未就緒就直接關瀏覽器）
     仍然可以被清掉，這是修正手段不是要擋的漏洞。前端同步處理：離線＋未就緒的客服，
     「操作」欄不顯示按鈕、改顯示灰字「離線中，無法設為就緒」；離線但仍是就緒狀態的客服
     照樣顯示「設為未就緒」按鈕。伺服器端檢查是必要防線（這支 API 本來就沒有角色權限
     檢查，任何人都能直接呼叫，光靠前端藏按鈕擋不住繞過 UI 直接打 API 的狀況）。

- **2026-07-21（續4）** 使用者反映客戶傳的 LINE 貼圖在主控台只顯示成純文字「[sticker]」，
  要求正常顯示。改法：
  - `LineWebhookServlet.handleEvent()` 的「message」分支，非文字訊息這一段以前一律存
    `"[" + msgType + "]"` 純文字佔位；現在 `sticker` 類型額外把 `packageId`/`stickerId`
    存成 `[sticker:packageId:stickerId]` 這個特定格式（其他非文字型態如圖片/影片/語音/
    位置**仍然維持**純文字佔位，這次沒有一併處理，範圍只限貼圖，YAGNI）。
  - `LineChatConsoleServlet` 的 `PAGE_HTML`／`loadMessages()`：用正規表達式
    `/^\[sticker:\d*:(\d+)\]$/` 偵測這個格式，命中就換成 `<img src="https://
    stickershop.line-scdn.net/stickershop/v1/sticker/{stickerId}/android/sticker.png">`
    （LINE 貼圖的公開 CDN，非官方文件記載但業界聊天機器人廣泛沿用，不需要
    `channelAccessToken`，已用 `curl` 實測回 200 + `image/png`）；氣泡樣式額外加
    `.bubble.sticker`（透明背景/無邊框，貼近 LINE App 原生貼圖不帶對話泡泡的樣子）；
    `img.onerror` 有 fallback（載入失敗就退回顯示「[貼圖]」純文字，不會變成破圖示）。
  - **⚠️ 舊訊息不會回溯顯示**：這個修法只對**部署後新收到**的貼圖訊息有效——已經存在
    `TcChatMessage` 裡、格式還是舊版純文字 `[sticker]` 的歷史紀錄，因為沒有存
    packageId/stickerId，補不回來，會繼續顯示成文字，需要客戶重新傳一次貼圖才會用新格式
    存、才能正常渲染。
  - **⚠️ 這次部署踩到自己訂的安全規範**：部署前的「確認無進行中對話」檢查與
    `docker restart` 兩個指令包在同一個 bash 呼叫裡執行、沒有先看檢查結果就繼續，結果
    當下其實有一房真實在進行中的對話（`🐶James`／`FChatId=U6efbfa5...`，是使用者自己在
    測試貼圖顯示問題用的帳號，`TcAcdAssignment` 顯示指派給 HCH，訊息記錄裡確實有一則
    `[sticker]` 純文字訊息，時間點正好對得上使用者提出這次需求的原因）——重啟後房間與
    指派紀錄都還在（DB 狀態不受 Tomcat 重啟影響），但嚴格來說**這次違反了自己訂的
    「重啟前必須先確認無進行中對話、再動手」流程**，只是這次剛好是使用者自己的測試帳號、
    影響有限。**教訓：這個檢查步驟務必獨立一個指令執行、看到結果是 0 才進下一步**，不要
    跟 `docker restart` 包在同一個 bash 呼叫裡, 那樣等於沒檢查。
  - **驗證方式**：用假測試帳號（先傳文字「真人客服」升級成真人房間、再傳一則模擬貼圖
    webhook 事件）確認 `TcChatMessage.FContent` 真的存成 `[sticker:11537:52002734]` 格式；
    `curl` 直接測 CDN 網址回 200。**這次收尾同樣遇到 Playwright 接管的 Chrome 卡住逾時**
    （`browser_tabs`/`browser_navigate` 都逾時，跟前幾次一樣的已知狀況），沒能在瀏覽器裡
    肉眼確認貼圖圖片真的渲染出來——只驗證到資料格式跟 CDN 網址本身，UI 呈現之後應該找
    時間補測。

- **2026-07-21（續5）** 兩個小修正，都是 Playwright 實測（這次終於順利接管成功）確認過：
  1. **RWD 補測確認生效**：上一輪部署後 Chrome 一直卡住沒能肉眼驗證，這次順利截圖確認——
     嵌在側邊選單旁的窄內容區裡，表格正確變成一張一張卡片（每格自帶欄位名稱），不再是
     擠壓換行的樣子，`@media (max-width:640px)` 那組 CSS 確認有生效。
  2. **「離線的就不要顯示了」**：`handleAgents()` 把「未就緒＋離線」的帳號直接濾掉不回傳
     （這是最常見、純雜訊的狀況）。**刻意沒有濾掉「離線但狀態仍是 Ready」的異常帳號**——
     這種帳號實際上仍占著輪流順位、`AcdAssignmentHome` 的派工演算法只看 `FStatus='Ready'`
     不檢查在不在線，藏起來會讓主管看不到「有人明明不在線卻還在吃新案子」這個真正該處理
     的問題。目前尚未實際觸發過這個例外分支被畫面顯示出來（現有測試帳號沒有剛好卡在這個
     狀態），邏輯上是對的但這個特定分支還沒有肉眼驗證過畫面呈現，之後遇到再確認一次。
     這次部署前的「進行中對話」檢查有確實獨立執行、看到房間仍開著但已閒置 9 分鐘才動手，
     沒有重蹈上一輪「檢查跟重啟包在同一個指令」的覆轍。

- **2026-07-21（續6）** 使用者不滿足於上一輪「離線的畫面上不顯示」，進一步要求「離線了他的
  就緒狀態也要清除掉」——底層 `TcAcdAgent.FStatus` 真的要同步改回 NotReady，不能只在
  `AcdAdminServlet` 讀取時做顯示層過濾。新增 `AcdAgentHome.syncOfflineAgentsToNotReady()`
  （`UPDATE TcAcdAgent SET FStatus='NotReady' WHERE FStatus='Ready' AND FAccountId NOT IN
  (SELECT FAccountId FROM TsOnlineUser)`，同時提供吃既有 `Connection`／自己開連線兩個
  重載），掛在兩個關鍵進入點：
  1. **`AcdAgentHome.lockNextReadyAgent(cn)` 開頭**——這是唯一真正決定「新客戶派給誰」的
     地方（`AcdAssignmentHome.dispatch()`/`enqueueAndDispatch()` 都會呼叫到），在同一個交易
     內先清掉離線但卡在 Ready 的帳號，確保**派工演算法本身**不會挑到一個不在線的人，
     不是只有畫面好看。
  2. **`AcdAdminServlet.handleAgents()` 開頭**——不用等下一次真的有新客戶進線觸發
     `dispatch()` 才會被清掉，主管開著管理台看的當下就是即時正確的狀態。
  - 上一輪加的「離線+顯示過濾」邏輯保留當第二道防線（理論上不會再被觸發，因為讀
    `TcAcdAgent` 之前就已經同步過），註解更新說明現在的主要機制已經是後端同步，不是
    畫面過濾。
  - **驗證**：手動把離線帳號 HCM 的 `TcAcdAgent.FStatus` 直接改成 `Ready`（模擬「忘記按
    未就緒就關瀏覽器」的情境），確認 `TsOnlineUser` 裡確實查不到這個帳號；用假 webhook
    觸發一次真實的「真人客服」轉接流程，確認：(1) HCM 的 `FStatus` 自動被同步回
    `NotReady`；(2) 這通對話最終正確指派給另一位**真的在線**的就緒客服（系統管理員），
    沒有誤派給已離線的 HCM。全程繞過瀏覽器直接用 curl/DB 驗證——因為驗證途中瀏覽器那邊
    出現使用者自己主動登出的畫面（`jumpCode=Logout`），判斷可能是使用者正在操作，主動
    停止繼續用 Playwright 動那個 session，改用 webhook 模擬走完整條派工鏈路驗證，沒有再
    嘗試重新登入去操作使用者的瀏覽器。

- **2026-07-21（續7）** ⚠️ **重大：Fortinet 第二次攔截 api.line.me，真實客戶對話被卡住，
  LINE API 連線模式改成 runtime 可切換的開關**。使用者回報「聊天框不能發訊息回答」，查證
  發現是真的有一位真實 LINE 客戶（`U70fd71bcb3bb6136af7e2197f645271d`）在等真人客服回覆，
  但不管 AI 自動回覆還是客服手動送出，一律在 TLS 憑證驗證這關失敗：
  ```
  PKIX path building failed: SunCertPathBuilderException: unable to find valid
  certification path to requested target
  ```
  用 `openssl s_client -connect api.line.me:443` 直接驗證，拿到的憑證是
  `subject=O=Fortinet, CN=Fortiguard SDNS Blocked Page`——跟 2026-07-20 那次一模一樣的
  Fortinet DNS/SSL 攔截，只是這次是**第二次發生**，證實封鎖是會反覆出現的常態，不是
  一次性事件。**同時發現一個獨立的既有 UX 缺陷**：`LineChatConsoleServlet` 的
  `send()` JS 呼叫 `fetch()` 後完全沒檢查 `response.ok`，送出失敗（無論什麼原因）
  UI 上完全沒有任何錯誤提示、只會靜默把輸入框清空，跟「訊息送出成功」長得一模一樣，
  這也是使用者一開始完全看不出來哪裡壞掉的原因之一（超出這次修復範圍，尚未修，
  之後應該補上失敗提示）。

  **修法：不再硬編碼「一定直連」或「一定走代理」二選一，改成 runtime 可切換的開關**：
  - `LineIntegrationHome` 新增 `useSocksProxy`（volatile boolean，預設 false）+
    `socksProxyHost`/`socksProxyPort`（預設 `host.docker.internal:8889`，可用
    `line.api.useSocksProxy`/`line.api.socksProxyHost`/`line.api.socksProxyPort` 三個
    properties 覆蓋初始值），`rawHttpsCall()` 依開關分流：開＝`new Socket(new Proxy(...))`
    + `InetSocketAddress.createUnresolved(host,port)`（主機名稱交給 proxy 端解析，
    唯一能繞過本地 DNS sinkhole 的寫法）；關＝原本的直連寫法。開關即時生效，不需要
    重啟 Tomcat。
  - `AcdAdminServlet` 新增「LINE API 連線模式」狀態列（頁面最上方）＋
    `GET/POST /admin/lineproxy`：顯示目前直連/走代理、對外目標，一顆按鈕切換；切到
    走代理前跳確認框，提醒「先確認 Windows host 的 SSH SOCKS 隧道還活著」（開關本身
    不建立/檢查隧道，那是主機端要另外維護的獨立行程，隧道沒開會導致
    `pushLine()` 丟 `Malformed reply from SOCKS server`，跟這次的 PKIX 錯誤是不同症狀，
    別搞混）。
  - **恢復隧道本身**：意外發現 `ssh.exe`（PID 對應）**已經在監聽 8888**——不確定是誰、
    什麼時候起的（可能是先前某次操作殘留、或使用者自己起的），新開一條反而
    `bind: Address already in use`。直接沿用這條既有隧道，用
    `curl --socks5-hostname 127.0.0.1:8888 https://api.line.me/v2/bot/info` 測過回
    HTTP 401（代表鏈路打通，只是 token 假的）。**教訓：啟用代理前一定要先探測 port
    有沒有現成在監聽，不要假設「skill 文件說隧道已移除」就真的沒有殘留行程**。
  - `netsh interface portproxy add v4tov4 listenport=8889 ... connectaddress=127.0.0.1`
    用 `Start-Process netsh -Verb RunAs` 觸發 UAC 補上（先前這條規則已經不在了）。
  - **驗證**：切換開關前後各發一次假 webhook 訊息比對 log——切之前是
    `PKIX path building failed`（被攔截），切之後變成
    `LINE API HTTP 400: {"message":"The value for the 'userId' parameter is invalid"}`
    這種**LINE 官方 API 真正回應的應用層錯誤**（因為測試帳號不是真的 LINE userId 格式），
    證實 TLS 連線已經成功穿透到真正的 LINE 伺服器，不再被 Fortinet 擋下來。**這次沒有
    直接對著真實客戶的房間送測試訊息**（不想在不知道要回什麼的情況下亂塞內容到真人
    對話串裡），改用假測試帳號驗證鏈路本身，實際要回覆這位客戶什麼內容留給真人客服
    自己決定。
  - **重啟時機**：這次重啟是在明知有真實客戶對話卡住等待的情況下執行的，且正是這次
    修復本身要解決的問題，判斷屬於使用者已經表達過的明確意圖，沒有再另外確認一輪就
    直接動手部署。

- **2026-07-21（續8）** 使用者反饋「服務完畢的房間就直接關閉了，不用隱藏」——`handleRooms()`
  原本會把已結束（`FCloseTime` 有值）的房間留在清單裡 24 小時，靠前端整組手動「×」
  隱藏（`localStorage`）機制讓客服自己清掉，這正是 2026-07-20 那次「全部隱藏後畫面空白
  看起來像資料遺失」事故的根源。改法：`handleRooms()` 的 SQL `WHERE` 條件從
  `FCloseTime IS NULL OR FCloseTime > NOW()-24hr` 簡化成單純 `FCloseTime IS NULL`，
  已結束的房間直接不會回傳、清單上馬上消失；`ORDER BY` 拿掉「未結束優先」排序（反正
  回傳的一定都是未結束），JSON 也不再帶永遠是 `false` 的 `closed` 欄位。前端把整組
  「隱藏」相關的死碼一併拿掉：`dismissed`/`loadDismissed()`/`saveDismissed()`/
  `dismissRoom()`/`showAllDismissed()`、`#hiddenHint` 提示與其 CSS、`.dismiss` × 按鈕、
  `.room.closed` 樣式、`closedRooms` Set 追蹤全部刪除；`closeRoom()` 成功後改成清空
  `currentRoom`（回到「請選擇左側房間」），因為房間本身已經直接從清單消失，不再只是
  隱藏 composer。**驗證方式**：這次容器 CDP（當時的舊 debug Chrome 設定）連不上，沒能走瀏覽器實測；改用直接
  對 DB 灌兩筆合成測試房間（一筆 `FCloseTime` NULL、一筆 1 小時前結束）跑部署後那支
  servlet 裡完全相同的 SQL 字串，確認只有未結束那筆被回傳、1 小時前結束的（舊邏輯
  24 小時內還會顯示）已被正確排除，測完立刻刪除測試資料；另外確認編譯零錯誤、部署後
  `docker logs` 無新例外。**编譯坑**：這次在 Git Bash 用 `javac -cp` 組多個分號分隔的
  classpath 路徑，若用 `/c/Users/...` 這種 POSIX 風格路徑直接嵌進同一個字串參數，
  Windows 原生 `javac.exe` 完全讀不懂（MSYS 只會自動轉換「看起來像獨立路徑」的單一
  參數，不會轉換嵌在一個分號字串裡的每一段），導致所有 import 都報
  `package XXX does not exist`——即使檔案路徑完全正確也一樣。**修法**：組 classpath
  前先用 `cygpath -w` 把每個路徑轉成 `C:\...` 格式再拼接分號字串，同樣的坑之後編譯
  任何東西都要記得。

- **2026-07-21（續9）** ACD 管理台「進線指派紀錄」加清除功能 + 「手動改派」欄位間距太窄的樣式修正。
  使用者反饋：(1) 紀錄應該可以被清掉，(2) 下拉選單跟改派按鈕黏太近。`AcdAdminServlet.java`：
  - **只允許刪除 `FStatus='Closed'` 的紀錄**（刻意的安全限制，不是遺漏）——`Waiting`/`Assigned`
    是 `AcdAssignmentHome` 派工引擎依賴的即時狀態，刪掉會讓進行中的房間變成孤兒指派（房間還開著
    卻查不到指派給誰）。後端 `handleDeleteAssignment()`/`handleClearAssignments()` 的 SQL
    `WHERE` 都帶 `AND FStatus='Closed'`，就算繞過前端直接打 API 也擋得住。
  - 新增兩個路由：`POST /admin/assignment/delete`（body `{id}`，`id` 是 `TcAcdAssignment.FId`
    真正的主鍵，不是 `FChatRoomId`——`handleAssignments()` 的 SQL 跟 JSON 一併補上 `id` 欄位）、
    `POST /admin/assignment/clear`（整批刪光所有 `Closed` 紀錄）。
  - 前端：「進線指派紀錄」標題旁加「清空已結束紀錄」按鈕（`.sectionHead` flex 容器）；表格最後
    一欄從「手動改派」改名成「操作」，`Closed` 狀態的列顯示「刪除」按鈕，其他狀態列維持原本的
    選客服下拉＋改派按鈕，兩者都包進新的 `.rowActions{display:flex;gap:8px}` 容器裡（不是直接
    在 `<td>` 上設 flex，因為手機版 RWD 的 `@media(max-width:640px)` 靠 `td{display:block}` 做
    卡片式排版，若 inline style 直接設在 td 上 flex 的 specificity 會蓋掉那條 media query 規則）
    ——這個 flex+gap 容器同時解決了「按鈕黏太近」的樣式問題，不用額外調間距。
  - **部署時機**：部署當下 DB 查到 `open_rooms=1`（使用者正在跟真實客戶 James 對話中），照使用者
    明確指示直接重啟（「影響不大」），沒有等對話結束。重啟＋`docker logs`確認無新例外、四個端點
    curl 測試（頁面 200、三個 API 未帶 session 皆正確回 401）驗證通過。

- **2026-07-21（續10）** 「報表中心」原本掛的是 Quicksilver 內建 `Qs.Report.Center` 報表引擎，
  使用者要求「把他改成聊天框產生相關的資料」。**先查證過原生引擎不可行**：`TsReport` 表在這個
  環境 0 筆、從未建過任何報表定義；深入點開「報表設定」（`003.002.003.002`，這個 EntityList
  隱藏在一個 `FEnabled` 為空但admin角色仍會忽略隱藏判斷的 outlook-bar 分區「進階設定」底下，
  要先在 DOM 裡直接 `.click()` 那個 `JuiOutlookBarItemHeadText` 才會展開，一般點擊點不到因為
  節點在可視區域外）發現：(1) 欄位設計預期要另外準備一份「範本檔」（類似 JasperReports 樣板），
  (2) 清單頁面完全找不到「新增」按鈕。使用者確認後改用已驗證有效的自建 servlet 模式（比照
  LINE聊天框/ACD管理台）。
  - 新增 `ChatReportServlet`（`com.chainsea.ecp.acd` package，跟 `AcdAdminServlet` 同目錄），
    掛在 `/aipower/acd/report`，三個資料端點：
    - `/report/inflow`：今日/近7天/近30天進線數、進行中vs已結束、近14天每日趨勢（`TcChatRoom`
      依 `FCreateTime`/`FCloseTime` 統計）。
    - `/report/agents`：客服績效——只算 `TcAcdAssignment.FStatus='Closed'` 且
      `FAssignTime`/`FCloseTime` 都非空的紀錄，平均處理時長 = `FCloseTime - FAssignTime`（從
      指派給該客服那刻算起，不是從建立房間算起——建立到指派之間是排隊等待時間，不算這位客服的
      處理時間）。
    - `/report/content`：訊息總量、平均每通對話訊息數、純文字/貼圖/照片比例（用
      `LineChatConsoleServlet` 既有的 `[sticker:...]`/`[image:...]` 標記字串判斷，跟主控台
      渲染邏輯共用同一套格式依據）。
  - 前端是純 HTML/CSS/JS 卡片+長條圖+比例條，沒有引入任何圖表函式庫（手刻 `<div>` 高度模擬長條
    圖），跟這個環境其他自建頁面風格一致，15 秒輪詢自動更新。
  - **`LineIntegrationStartupListener.java`**（唯一一份存在於 `line-integration/src/`，
    `acd-integration/src/` 沒有這個檔案——兩個資料夾目前是有落差的並行副本，之後若要修改共用檔案
    要注意去哪個資料夾找）新增 `ChatReportServlet` 的 `addServlet` 註冊區塊
    （`setLoadOnStartup(6)`，接在 `AcdAdminServlet` 的 5 之後）。
  - **TsMenu**：「報表中心」那筆（`FId=e7a28f62-d8d0-4737-bb0a-87e9ff7f2329`）原本
    `FPageId` 指向內建 `Qs.Report.Center`，改成 `FPageId=NULL`、
    `FExternalPageUrl='/aipower/acd/report'`（同源相對路徑，沿用之前 iframe session cookie
    的教訓），改前已 `mariadb-dump --where` 備份整筆。
  - **部署時機**：這次改動含 `LineIntegrationStartupListener`（`ServletContextListener`，只有
    容器啟動時才會執行 `addServlet`，不能像單一 `.class` 覆蓋那樣靠 `docker cp` 就生效，必須
    `docker restart`），部署前確認 `open_rooms=0`（沒有真實客戶在等）才動手。
  - **驗證**：四個端點 curl 測試（頁面 200、三個 API 未帶 session 皆 401）、`docker logs` 無新
    例外；接著用 Playwright 實際登入畫面截圖確認三個區塊都正確渲染出真實資料（今日進線4／近7天
    5／近30天5／已結束5，14天趨勢長條圖，訊息總量47、純文字40/貼圖5/照片2 的比例條）——「客服
    績效」表格顯示空白是正確行為，因為 `TcAcdAssignment` 當下剛好是 0 筆（前一個改動測試「清空
    已結束紀錄」按鈕留下的狀態），不是這次改動的錯誤。
  - **過程中的帳號切換**：因為 HCH 的「業務」角色在側邊欄完全看不到「設定」分區（無此角色權限），
    為了摸清「報表設定」在選單樹的位置，暫時登出 HCH、改用 administrator（見上方密碼變更記錄）
    登入這個 debug Chrome 視窗探索，事後沒有再切回 HCH——這個 debug 視窗目前是系統管理員身份，
    跟使用者自己平常用的瀏覽器/帳號是分開的 profile，不受影響。

- **2026-07-22** ✅ **新增 `Ecp.Asset`——資產管理標準六大核心 Unit，供外部 Flutter client
  CRUD 呼叫**。使用者要做一個資產管理（IT/實物設備）的 Flutter Windows client，backend
  從零建置。跟 MikoPBX/ACD 系列一樣走標準六大核心（`com.chainsea.ecp.asset` package，
  `TcAsset` 表，`FId=393268d2-d477-4a36-9ab2-403eebd21085`），掛在「開發→工具」底下
  （`004.002.008`）。原始碼與建置 SQL 存在本 skill 目錄
  `asset-integration/`（`src/`、`asset_unit.sql`），照抄改個 UUID/package 名稱就能建
  下一個純 CRUD（無外部 API 依賴）的 Unit。
  - **保管人帳號名稱解析**：`AssetServiceImpl` override `doCreate`/`doUpdate`（用
    `javap -p -c` 反編譯 `EntityServiceImpl` 確認兩者簽章：`doCreate` 回傳 `UUID`、
    `doUpdate` 回傳 `void`），讀取 payload 裡一個不對應任何 DB 欄位的
    `FCustodianAccountName` key（`Record(JSONObject)` 建構子本來就會把所有 JSON key
    存進內部 Map，不限於已註冊的 `TsField`），查 `TsAccount.FLoginName` 換成真正的
    `FCustodianAccountId` 再寫入；帳號不存在直接拋例外、整筆記錄不會寫入（沿用
    `Ecp.MikopbxSetting` 那次 `doUpdate()` override 驗證過的「前置邏輯失敗就不呼叫
    super」保護模式）。
  - **✅ 首次完整反編譯確認框架 REST CRUD 端點的真實參數格式**（過去幾次 MikoPBX/ACD
    只確認了 `save.data` 的 `{"data":[...]}` 陣列包物件，這次補齊了其餘三個）：
    - `getListData.data`：POST body 可以是 `{}`，回應
      `{"data":{"records":[...],"pageIndex":N,"hasNextPage":bool,"timeCost":N}}`，
      `records[]` 只含 `TsListField` 裡設定的摘要欄位（本例是 `FAssetCode`/`FName`/
      `FCategory`/`FStatus`/`FLocation`/`FPurchaseDate`，值是 `null`/未設定的欄位不會
      出現在物件裡）。
    - `getItem.data`：參數 key 是 `entityId`（單數，不是 `id`），回應是包含全部欄位的
      扁平物件。
    - `delete.data`：參數 key 是 `entityIds`（陣列），成功回應 `{}`。
    - `save.data` 成功回應是 `{"entityIds":["<uuid>",...]}`（不是單一 `FId`）。
    - 錯誤回應統一格式：`{"_failed":true,"stackTrace":"","message":"..."}`。
    - 登入端點 `Qs.OnlineUser.login.data`（用 `javap` 反編譯
      `OnlineUserActionImpl.login()` 確認），必要參數除了 `loginName`/`password`還有
      `language`（缺這個會回 `"Language 'null' is invalid."`），這個系統帳號的語言碼
      格式是小寫連字號 `zh-tw`（查 `TsAccount.FLanguage` 實際值得到，不是猜的）。
  - **⚠ 修好 `scripts/deploy-servlet.sh` 的一個真實 bug**：腳本原本直接對每個檔案
    `docker cp <本機class> <容器:目標路徑>`，如果目標的 package 目錄在容器裡是**全新的**
    （像這次 `com/chainsea/ecp/asset` 從未存在過），`docker cp` 對不存在的巢狀目錄會
    直接失敗 `Could not find the file ... in container`——過去所有整合都是在已存在的
    package（`com/chainsea/ecp/lineintegration`、`mikopbxextension`、`acd` 等）底下加
    檔案，沒踩過這個坑。修法：`docker cp` 前先加一行
    `docker exec ... mkdir -p $WEBAPP/classes/$pkg`，已測試確認全新 package 也能一次
    部署成功。**這個修正是永久的**（改在腳本本體，不是這次部署現場繞過），下次任何
    全新 package 的整合都不會再踩到。
  - **Registry 是 lazy 註冊，不是重啟當下註冊**：以為重啟後
    `docker logs | grep "Register unit: Ecp.Asset"` 就能驗證 Unit 有載入，結果重啟後
    完全沒有這行 log（也沒有任何錯誤）。查證發現連早就穩定運作的
    `Ecp.MikopbxSetting` 也是同樣情況——它的「Register unit」log 最後一次出現是
    2026-07-18，之後容器明明重啟過很多次都沒有再打印。**真正的結論：這個框架版本的
    Unit/Dao/Service 是在第一次被實際呼叫時才透過 Registry 建立 proxy
    （log 訊息其實是 `Create proxy: Qs.Xxx.Dao` 這類，不是重啟時全表掃描註冊），
    重啟後單靠 log 完全看不出新 Unit 到底能不能用**，唯一可靠的驗證方式是直接呼叫
    一次真實的 CRUD 端點（跟本次 Task 7 的做法一致）——這條經驗對之後任何新 Unit 的
    部署驗證流程都適用，不要再假設「重啟後查 log 就能確認」。
  - **驗證**：完整 curl CRUD 循環（新增→查清單→改保管人成功案例→改保管人失敗案例→
    刪除→查清單確認消失→直接查 DB `TcAsset` 交叉確認真的刪除）全部通過，過程全程
    無關聯例外。

- **2026-07-22（續）** ✅ **從零建立一個低權限「服務帳號」（只給特定 Unit 的部分 CRUD
  權限、給程式寫死密碼用）——完整踩坑記錄，之後任何類似需求直接照這份做**。背景：
  Flutter client 需要一個不用人在旁邊登入、背景自動呼叫 API 的身份（`asset-agent`），
  只給 `Ecp.Asset` 新增/查看/修改，不給刪除。

  **角色（Role）+ 權限（Privilege）→ 用 UI 建角色本體，用 SQL 補權限**：`Qs.Role.List.page`
  新增「通用角色」（UI 表單只有名稱/描述，沒有 Unit 級權限勾選格——這個框架版本的角色
  表單沒有直接勾 Unit 權限的介面，要另外處理，見下一段），存檔後直接對
  `TsRolePrivilege` INSERT 三筆（`FRoleId`=新角色、`FPrivilegeId`=目標 Unit 的
  `TsPrivilege.FId`，來源可以直接查
  `SELECT FId,FPrivilegeTypeId FROM TsPrivilege WHERE FUnitId='<目標UNIT_ID>'`）。

  **帳號（Account）本身只是登入憑證，不含任何權限**：`Qs.Account.List.page` 新增帳號
  只會產生 `TsAccount` 一筆（登入名/密碼/語言），**光有這筆帳號 `Qs.OnlineUser.login.data`
  會回 `{"code":"Qs.Account.NoIdentity","message":"帳號至少需要一個身份。"}`**——帳號
  必須至少掛一個「身份」才能登入，這是框架強制要求，不是可選功能。

  **身份鏈的真實結構（反查 `TsAccountIdentity`/`TsRoleUser` 資料才搞懂，不是文件寫的）**：
  ```
  TsAccount (登入憑證)
    ← TsAccountIdentity.FAccountId
      → TsAccountIdentity.FIdentityTypeId (只有兩種：職務/員工，都指向同一張 TsUser 表)
      → TsAccountIdentity.FEntityId = TsUser.FId (「員工」實體，不是帳號本身)
          ← TsRoleUser.FUserId = 這個 TsUser.FId（不是 TsAccount.FId！）
            → TsRoleUser.FRoleId = TsRole.FId
  ```
  `TsRoleUser.FUserId` 對的是「員工」（`TsUser`）不是「帳號」（`TsAccount`），這是最容易
  搞錯的地方——查 `TsIdentityType` 只有 `職務`(564cf69e-76d6-4baf-b584-6e04c2911dae)/
  `員工`(ea960b69-1c3f-4719-9482-f9ebe0264c38) 兩種，兩者的 `FUnitId` 都指向
  `Qs.User`/`TsUser`，代表沒有「純角色、不掛員工」這種身份類型可選——**要讓一個帳號能登入
  並擁有角色權限，一定要建一筆 `TsUser`（員工）記錄**，即使這個「員工」根本不對應真人。

  完整建立順序（服務帳號用，SQL 直接寫，不透過 UI 走「身份」分頁裡的「新增」——那個是
  一個巢狀 iframe 彈窗選 `身份類型`+`關聯物件`，`關聯物件` 選擇器要求先有 `TsUser`
  記錄才選得到，兩者互為前提，直接 SQL 插三張表最乾脆）：
  ```sql
  INSERT INTO TsUser (FId, FName, FAccountId, FEnabled, FLoginName, FIndex)
    VALUES (@USER_ID, '<顯示名稱>', @ACCOUNT_ID, b'1', '<登入名>', 99);
  -- FDepartmentId 雖然 schema 允許 NULL，但登入流程實測要求非 NULL，否則卡在
  -- {"code":"Qs.Entity.NotExistById","message":"ID 為“null”的部門不存在。"}
  -- 隨便指一個真實存在的頂層部門即可（SELECT FId FROM TsDepartment 找一筆）：
  UPDATE TsUser SET FDepartmentId='<某個真實 TsDepartment.FId>' WHERE FId=@USER_ID;

  INSERT INTO TsAccountIdentity (FId, FName, FAccountId, FIdentityTypeId, FEntityId, FDefault, FIsMainDuty, FIndex)
    VALUES (@IDENTITY_ID, '<顯示名稱>', @ACCOUNT_ID,
            'ea960b69-1c3f-4719-9482-f9ebe0264c38'/*員工*/, @USER_ID, b'1', b'0', 1);

  INSERT INTO TsRoleUser (FRoleId, FUserId) VALUES (@ROLE_ID, @USER_ID);
  ```

  **⚠ 帳密雜湊要用 `PasswordUtil.encode()` 一模一樣的演算法**（見本檔案上方
  `scripts/reset_admin_password.ps1` 章節）：8-byte salt + `"secret"` + 明文，SHA-256 疊代 1024
  次，自訂 Base16（a-p）編碼，`FPassword` 存 `salt(8)+digest(32)` 共 80 個字元。**新帳號
  的 `FEnabled`（bit 型別）用 `mariadb` 指令列直接 `SELECT` 出來看起來像空字串**，不代表
  沒設定成功——bit(1) 的值 1 在終端機顯示是不可印字元，要 `SELECT FEnabled+0` 轉數字才
  看得出真正的 0/1。

  **⚠ 改完帳號/身份/角色相關資料，不 `docker restart` 一定測不出真結果**——這是繼
  TsMenu/TsSystemParameter 之後第三次證實這個框架把帳號/身份查詢也快取在 JVM 記憶體，
  純 SQL UPDATE 完不重啟，`Qs.OnlineUser.login.data` 會卡在莫名其妙的中途錯誤（本次
  實測是卡在部門查詢那步，一開始以為是 `FDepartmentId` 沒設對，其實是舊 session/
  快取殘留，設對 `FDepartmentId` 後**還是**要重啟才生效）。

  **⚠ 全新角色的權限最容易漏掉的一步：`TsRolePrivilege.FGlobal` 必須明確設成 `b'1'`**
  （不能留 `NULL`）——反編譯 `com.jeedsoft.quicksilver.privilege.Privilege.
  hasCreatePrivilege()`（`quicksilver-module-main-7.2.2.jar`）證實：`FGlobal` 不是
  「這筆權限存不存在」的旗標，是「走哪一條檢查路徑」的分歧點——`isGlobal()` 為
  true 走簡單的全域檢查（`hasGlobalPrivilege`），為 false/NULL 會落到
  `hasCreatePrivilegeByIdentity()` 這條**依賴「部門/擁有者」欄位比對**的路徑，
  對一個像 `TcAsset` 這種沒有部門歸屬欄位的普通業務表，這條路徑會直接判定沒有權限，
  回 `{"code":"Qs.Privilege.CannotCreate","message":"您無權建立該資產管理。"}`——
  即使 `TsRolePrivilege` 那筆資料明明存在、`FPrivilegeId` 也對，一樣會被擋。
  **`FGlobal=NULL` 在 `Ecp.MikopbxExtension` 等既有 Unit 的 `TsRolePrivilege` 裡從沒
  出過問題，是因為那些測試全程只用 `administrator`（`職務` 身份類型）測過，
  administrator 大機率有另一條 superuser 捷徑跳過這整套檢查，從來沒有真正被走到
  identity-scoped 這條路径**——這代表本系統過去所有「用 administrator 測過 CRUD 就
  算驗證完畢」的結論，都只驗證了「有 Unit 本身可以動」，**沒有驗證過 TsRolePrivilege
  的 FGlobal 欄位語意**，之後若要幫任何真實限權角色（非 admin）開權限，
  `FGlobal` 一律明確設 `b'1'`，不要複製既有範本裡的 `NULL`。

  **✅ 完整驗證方式**：專門寫一支測試 curl 序列，用新帳號密碼登入 → 依序打
  `save.data`(create)/`getListData.data`/`getItem.data`/`save.data`(update)/
  `delete.data`，確認前四個成功、`delete.data` 正確回
  `{"code":"Qs.Privilege.Check2","message":"您沒有“Xxx”的“刪除”許可權。"}`
  ——光測試「能不能登入」或「Unit 能不能用」不夠，要把「這個角色設計上刻意不給的那個
  操作」也測一次真的被擋下，才算驗證了權限设计本身（不是只驗證了帳號能動）。

  **建帳號/角色本身用 debug Chrome + Playwright 走 UI**（`chrome.exe
  --remote-debugging-port=9222 --user-data-dir=C:\Temp\ChromeDebug`，並接共用的
  `playwright-vrs` MCP server，見 `connect-chrome` skill），**但這個框架的多層巢狀 iframe 對話框用
  accessibility snapshot 的 ref 系統很不穩定**（同一個 ref 編號在下一次 snapshot
  常常物件已經不對應了，尤其是每次點擊都會讓所有巢狀 iframe 重新編號）——比較可靠的
  做法是用 `browser_evaluate` 搭配 `Array.from(document.querySelectorAll('iframe'))`
  逐層找到目標 iframe 的 `src`（含 `unitCode=Qs.Account`/`Qs.Role` 這類參數，比對得出
  正確的那一層），再用 `contentDocument.querySelectorAll('*')` 配文字內容比對找目標
  元素直接呼叫 `.click()`，比連續呼叫 `browser_click` 配 ref 可靠很多。**直接對
  `.page` 網址 `browser_navigate` 會弄壞 MainFrame 的內部路由狀態、觸發
  `?jumpCode=SessionInvalid`**——一定要透過點擊左側選單樹讓框架自己在 MainFrame
  內部開新的 iframe 分頁，不要自己組網址跳轉。

- **2026-07-23** ⚠ **服務帳號密碼被使用者直接在後台改掉，導致 Flutter client 端硬編碼的
  舊密碼連續打錯、快要把帳號鎖死**——`asset-agent` 帳號被使用者手動改成登入名 `agent`/
  密碼 `1234`，但 Flutter 端 `AutoReportService` 還在用舊的 `asset-agent`/
  `AgentPw#2026xR7q`，每 30 分鐘/每次開程式就打一次錯密碼，`TsUserInputPasswordErrorCount.
  FCount` 逼近鎖定上限（訊息「您還有 2 次輸入機會」）。**教訓**：任何寫死在外部
  client 程式碼裡的服務帳號密碼，一旦懷疑登入失敗，第一步永遠是先查
  `SELECT * FROM TsUserInputPasswordErrorCount WHERE FLoginName='<login>'`
  確認還剩幾次機會、**不要再用同一組憑證重試**去賭「這次應該對了吧」——這條規則
  `scripts/reset_admin_password.ps1` 那節已經寫過一次，這次是同一個坑在不同帳號上又踩了
  一遍，代表這類「client 端寫死密碼」的設計，密碼來源真相只有一份（DB），任何一邊
  改了都要同步另一邊，且改密碼前後都要留意鎖定計數。修法：直接用
  `scripts/reset_admin_password.ps1` 同款雜湊演算法重設成使用者指定的新密碼、清空鎖定計數、
  `docker restart aipower-app`、只驗證一次（不要連續嘗試），並回頭同步更新所有寫死
  這組憑證的地方（Flutter 原始碼、design spec、plan 文件、記憶檔）。

- **2026-07-23** ⚠️🔬 **深入調查「這套系統有沒有 API Key/Token 機制可以取代帳密」——
  架構完全查清楚了，但實際卡在 Registry 啟動流程沒有觸發，最終擱置、退回帳密方案**。
  使用者對 Flutter client 把 aipower 帳密寫死（即使已改成 DPAPI 加密存本機）仍有疑慮，
  一路追查「資料整合」選單下的 Token 設定/本地API/API 2.0，過程記錄在
  `ecp-action-authz-gap` skill（結論是 Token 設定/本地API/直連API組全部沒有真正生效
  的驗證邏輯），但使用者後來指出 D:\_暫時保留\ECP(全部+V) 這個備用安裝
  （`quicksilver-module-main-7.1.31.beta25.jar`）裡有本機 Docker 版本
  （7.2.2）缺少的 `ApiServiceImpl`/`ApiActionImpl` 實作，逼出了下面這段更深入的架構
  研究。

  **✅ 架構完全查清楚了（Api 2.0 / `Qs.Api` 子系統）：**
  - 這套系統其實有一個**跟六大核心平行的第七層**：`Api` 層（`BaseApi`/`EntityApi<T>`/
    `EntityApiImpl<T>`，位於 `com.jeedsoft.quicksilver.base.api`），透過
    `TsUnit.FApiClassName` 註冊，跟 Model/Dao/Service/Action/Home 的 Registry 註冊
    方式完全一樣（一樣要寫 `super(UNIT_ID, XxxModel.class)` 建構子）。
  - **這一層本機 Docker 部署（`aipower-module-base-7.3.12.5.jar` +
    `quicksilver-module-main-7.2.2.jar`）其實完整都在**，一開始判斷「缺 class 要跨版本借」
    是錯的——`com.chainsea.ecp.integration.service.impl.ApiServiceImpl`（Chainsea 客製版，
    30KB 真實實作）跟 `com.jeedsoft.quicksilver.integration.action.impl.ApiActionImpl`
    都已經在本機的 jar 裡，只是從來沒有任何 Unit 設定 `FApiClassName` 用過，才會誤以為
    是空的。**這個坑的教訓**：判斷「這個 class 缺不缺」一定要對照 `TsUnit` 實際指到的
    完整類別名稱直接 grep jar 檔案列表確認，不要只看某一個特定 jar（例如只查
    `quicksilver-module-main` 沒查 `aipower-module-base`）就下結論、更不要被自己
    unzip 遞迴萬用字元漏抓檔案的操作失誤誤導成「真的缺」。
  - **真正的認證機制**：`ApiServiceImpl.processToken()` 讀 `Authorization: Bearer
    <tokenId>` header（或 body 的 `tokenId` 欄位），呼叫
    `TokenService.access(ctx, tokenId)`（`Qs.Token` Unit，`TokenApiImpl.apply()`
    用帳密換來的）解析出真正的 `accountId`，設進 `ApiContext.setUserId(...)`——這才是
    「用一次性帳密換一個可獨立撤銷、不等於真密碼的憑證」這個需求真正對應到的框架設計。
  - **`EntityApiImpl<T>` 的 `getItem`/`getList`/`create`/`update`/`delete` 方法本身
    就已經標好 `@Api` 註解**（`javap -v` 反編譯確認 `RuntimeVisibleAnnotations` 有
    `Lcom/jeedsoft/quicksilver/integration/annotation/Api;`），子類別只要純繼承+建構子
    就會自動被掃描註冊，不需要覆寫——跟 Chainsea 自己既有的
    `com.chainsea.ecp.customer.api.impl.CustomerApiImpl` 等一大票業務模組的真實範本
    完全一致（`aipower-module-*.jar` 裡至少 20+ 個模組都用這個模式，值得注意
    `CustomerApiImpl` 選擇**覆寫**了 `getList`，原因見下面「路徑衝突」小節）。
  - **`@Api` 註解的 `value()`/`path()` 都不設就會用 `value` 當 fallback**（`path()`
    annotation default 是空字串 `""`，`ApiInfo` 建構子若 `path()` 空就退回用
    `value()`），`EntityApiImpl` 各方法的 `value` 分別是
    `item`/`list`/`create`/`update`/`delete`。
  - **⚠️ 註冊表的 key 是裸的 `ApiInfo.path`，完全沒有 Unit code 前綴**（反編譯
    `findApi()`/`loadApi(ServiceContext, UnitModel)` 的 `Map.put()` 呼叫確認，
    `key = ApiInfo.path`，不是 `unitCode + path`）——**這代表如果兩個不同 Unit
    都直接繼承 `EntityApiImpl` 不覆寫，會在全域共用的 `plainApiMap` 裡搶
    `"list"`/`"create"` 這種裸名字，第二個載入的會觸發
    `throwDuplicationException`**。這正是 `CustomerApiImpl` 明明可以純繼承、卻選擇
    覆寫 `getListData`（換一個不會撞名的新方法名+新 `@Api(value=...)`）的真正原因。
    **之後要幫任何 Unit 開 Api 層，如果预期这个环境会有多個 Unit 都掛 Api，一律要
    覆寫並給獨一無二的 `@Api(value="<unit小寫>/xxx")` 這類前綴路徑，不要依賴裸繼承**
    （這次因為 CRM 模組沒裝，Customer 這類會撞名的 Unit 事實上是 dormant 的，才沒有
    在部署 `AssetApiImpl` 空殼繼承版時炸出 duplication exception——但這只是運氣好，
    不代表可以依賴這個假設）。
  - **`TsUnit.FApiEnabled` 是跟 `FApiClassName` 分開的獨立開關**——只設
    `FApiClassName` 沒有把 `FApiEnabled` 設成 `b'1'`，`isApiEnabled()` 檢查會擋掉，
    這個 Unit 的 Api 完全不會被載入。**這是這次除錯過程中真正踩到、且已經找到答案的坑**
    （不像下面那個仍未解的謎團）。

  **❌ 卡住、未解的謎團**：即使 `FApiEnabled=1` 都設對（含原廠內建、本來就是
  `FApiEnabled=1` 的 `Qs.Token` 也一樣），**整個 `plainApiMap`/`wildcardApiMap`
  在這個部署裡完全是空的**——把 `com.jeedsoft.quicksilver.integration`、
  `com.chainsea.ecp.integration`、`ApiServlet` 全部開到 TRACE 等級（見下方「如何開
  debug log」），送請求後 `debug.log` 只看到
  `[LocalAPI] invocation start. path = apply`（`ApiServlet$Inner.process()`
  一律會印的起始訊息，不代表成功），完全沒看到 `loadApi()`/`findApi()`/
  `"[API] dispatch invocation"` 這些只在真正執行到對應程式碼時才會印的訊息——代表
  `ApiServiceImpl.loadApi(ServiceContext, List<UnitModel>)`
  這個「掃描所有 Api-enabled Unit、填進註冊表」的函式，**從容器啟動到現在完全沒有被
  呼叫過一次**，即使是原廠自己配置好的 `Qs.Token` 也一樣半途而廢。反編譯
  `com.jeedsoft.quicksilver.registry.Registry` 有看到它 reference 了
  `ApiHome.getService()`/`loadApi`/`"Initializing"` 字串，但實際呼叫的**觸發條件**
  是什麼（可能是另一個系統參數/授權層級判斷、也可能是走了另一條完全不同的啟動路徑）
  還沒查出來——**這是下一次要接續查的起點**：反編譯 `Registry` 呼叫
  `ApiHome.getService().loadApi(...)` 那一段前後的完整判斷邏輯，找出為什麼在這個部署
  完全沒被觸發。

  **如何開 debug log 除錯這類問題（一般性技巧，值得記住）**：設定檔在
  `/app/apache-tomcat/extension/aipower/config/log4j2.xml`（容器內），
  **`monitorInterval="30"` 表示改完直接 `docker cp` 進去、等 30 秒就自動生效，不用
  restart**；`<Loggers>` 區塊裡已經有現成、只是被註解掉的
  `Qs.Monitor.Api`/`LocalApiServlet`/`RemoteApiUtil` 等 logger 可以直接取消註解；
  對應的 log 檔在 `/app/apache-tomcat/extension/aipower/log/{debug,trace,info,
  warn,error}.log`（依等級自動分流，`debug.log` 最實用）。**用完一定要記得改回去**
  （這次已還原成原本只有 `Qs.Monitor.Thread` 開著的狀態），TRACE 等級長期開著會產生
  大量 log 檔案。

  **目前的最終決定**：Flutter client 維持用「首次啟動彈窗收集帳密、DPAPI 加密存本機」
  的方案（見上面「自動回報專用帳號」章節），**這個 Api2.0/Token 方案的程式碼
  （`asset-integration/src/com/chainsea/ecp/asset/api/`）保留在原地、`FApiEnabled`
  也保留開啟狀態**（無害、目前沒有任何呼叫路徑會用到它），先擱置，等有更多時間再回來
  查 `Registry` 啟動序列這個未解的部分。

- **2026-07-23（同日後續）** 🎯 **找到繞過 `loadApi()` 謎團的現成範例：`D:\Allen's\
  allen-test-local-api-1.0.zip` + `dashboard-simple.zip`**——這是另一位開發者
  （Allen）已經驗證可行、正在用的「外部頁面打本地資料」範例，證實根本不需要解開
  Api2.0 的 `loadApi()` 為何沒被觸發，因為他用的是**完全不同、本來就在運作的舊版
  「本地 API」(Local API) 機制**，而不是 `@Api` 註解那條路。

  **架構**（`dashboard-simple/README.md` 原文說明）：
  `功能表「外部頁面」→ dashboard.html → dashboard-app.js 打
  GET /aipower/openapi/demo/list?unitCode=Qs.User → DemoUnitListHandler（JAR）
  → 回 JSON`。**URL 一樣是 `/openapi/*`，一樣進 `ApiServlet`**——但因為 Api2.0
  registry（`plainApiMap`）是空的，`ApiServlet$Inner.process()` 的 `invoke()`
  回傳 false，流程就掉進既有的 fallback 分支 `invokeAsLocalApi(path)`：拿
  `path`（`demo/list`）去查 `TsLocalApi.FPath`，DB 裡設定這筆的
  **呼叫目標類型 = API Method、目標類別 = `com.test.allen.DemoUnitListHandler`、
  方法 = `execute`**，框架直接用反射呼叫這個方法——**這整條路徑完全不經過
  `ApiServiceImpl.loadApi()`，天生就不受那個未解之謎影響**。

  **反編譯 `allen-test-local-api-1.0.jar` 確認的兩種可用 handler 寫法**：
  1. `execute(HttpServletRequest, HttpServletResponse)` 靜態方法（`DemoUnitListHandler`
     用這個）——`javap` 反編譯看到它自己手動組
     `ActionContext`（`setRequest`/`setResponse`/`setLanguage("zh-tw")`/
     `setFromWebService(true)`），接著直接 `Registry.getService(unitCode)`
     取得 `EntityService`、`Criterias.create()` 查資料、`ListHome.getService()
     .getDataJson(...)` 轉成前端要的格式、自己 `writer.write(json.toString())`
     輸出——完全是手刻，不靠 `EntityApiImpl` 那套自動 CRUD。
  2. `implements com.jeedsoft.quicksilver.integration.type.LocalApiProcessor`，
     覆寫 `process(LocalApiContext)` 回傳 `LocalApiResult`（`TestLocalApiProcessor`
     用這個）——更貼近框架原生介面的寫法，`new LocalApiResult(ctx).putAll(json)`。

  **⚠️ 關鍵代價：這條路完全沒有框架內建驗證**。反編譯確認
  `DemoUnitListHandler.buildActionContext()` 裡面**寫死**
  `ctx.setUserId(UserHome.ADMINISTRATOR)` 並呼叫自己的
  `loadAdministratorIdentity()`（查 `AccountIdentityDao.getItems(ctx,
  ADMIN_ACCOUNT_ID, UserHome.ADMINISTRATOR)`，查不到就直接 new 一個空的
  `Identity` 塞 Administrator 的 entityId/accountId）——不管呼叫者是誰、有沒有
  帶任何憑證，這個 handler 一律用 Administrator 身份執行。**這印證了
  `ecp-action-authz-gap` skill 裡早先的結論：「本地 API」表單上的
  `FAuthType`（安全驗證方式，例如選 Token）下拉選單只是 UI 欄位，框架本身不會替
  你驗證任何東西——驗證與否、驗證什麼，全部要 handler 自己手動做**。

  **對 Asset 的 Token 需求怎麼落地（下次要做就直接照這個做，不用再等 `loadApi()`
  查出結果）**：寫一個 `AssetLocalApiHandler`（`execute(req,resp)` 簽名或
  `implements LocalApiProcessor` 皆可），在方法最開頭自己做：
  1. 讀 `request.getHeader("Authorization")`，剝掉 `"Bearer "` 前綴拿到
     `tokenId`；
  2. 呼叫 `TokenService.access(ctx, tokenId)`（`Qs.Token` Unit 既有的驗證邏輯，
     這步驟真的有效——已在更早的 Token 設定調查中反編譯確認）解出真正的
     `accountId`/`Identity`，**不要學 Allen 範例寫死 Administrator**；
  3. 解析失敗（token 過期/不存在）就回 401，成功才繼續往下用
     `AssetHome.getService()`/`Registry.getService(...)` 做 CRUD；
  4. 在後台「本地 API」表單掛一筆 `FPath=asset/xxx`、目標類別指到這個 handler。

  這樣就能達成「用一次性換來、可獨立撤銷的 token 取代 Flutter 端存密碼」的目標，
  且完全繞開 Api2.0 `loadApi()` 從未被呼叫這個仍未查出根因的深層問題。

- **2026-07-23（同日再續）** ✅ **上面的設計已經真的寫出來、部署、端到端測試通過，
  這條 Token 認證路徑正式可用**。原始碼：
  `asset-integration/src/com/chainsea/ecp/asset/integration/AssetLocalApiHandler.java`
  （Asset CRUD，直接呼叫既有 `AssetApiImpl` 的 `getItem/getList/create/update/delete`，
  一行 CRUD 邏輯都沒重寫）+ `LocalApiAuthHandler.java`（發 token / 撤銷 token，因為
  `Qs.Token.apply.api` 也是 Api2.0、一樣被 `loadApi()` 卡死，光有 Asset handler
  永遠拿不到第一個 tokenId，這個是補上的「用帳密換 token」入口）。過程踩了三個坑，
  都已經修進最終版程式碼，記錄下來避免下次重踩：

  1. **`TsLocalApiGroup`/`TsLocalApi` 的 `FRequestFormat`/`FResponseFormat` 必須是
     `'JSON'`（全大寫）**，寫成 `'Json'` 會在 `DefaultLocalApiRequestParser.parse()`
     直接 500（`RuntimeException: Unexpected LocalAPI request format: Json`）——反編譯
     確認程式碼寫死比對字串常數 `"JSON"`，不是大小寫不敏感比對，也不是任何下拉選單
     的枚舉值來源（這兩張表在這個部署裡本來就是空的，UI 上完全沒有既有資料可以參考）。
  2. **`ApiContext.setThreadInstance(ac)` 反編譯確認會無條件呼叫
     `ac.getServiceContext()`，傳 `null` 進去必定 NPE**——一開始學一般資源清理習慣在
     `finally` 裡呼叫 `ApiContext.setThreadInstance(null)` 想「用完歸零」，結果這個
     NPE 蓋掉了 try 區塊裡的真正例外，回應只看得到 `"ac" is null`，一度誤導成
     `dispatch()` 本身壞掉。**最終版直接拿掉這個 finally 清理**（Api2.0 真正的
     dispatch 路徑本身也沒有對應的清除呼叫，request-scoped 用完即丟即可）。
  3. **`OnlineUserService.login()`（`LoginOptions.setApiLogin(true)` 換來的）回傳的
     `LoginResult.getTokenId()` 是 `TokenUtil.createNewTokenId()` 的「新格式」
     session token（DB 對應 `TsOnlineUser.FTokenId`），不是 `Qs.Token` 子系統
     （`TsTokenConfig`/`TokenService.access()`）那組「應用程式 API Key」**——一開始
     誤用 `TokenService.access()` 驗證，永遠回 `null`（`Asset.Token.Invalid`）。反編譯
     `OnlineUserDaoImpl.optItem()`（SQL 是 `select * from TsOnlineUser where
     FSessionId=? or FTokenId=?`）才抓出真正該用
     `OnlineUserHome.getService().getItem(tokenId)` 查 `OnlineUserModel`、再拿
     `.getIdentity()`；撤銷則是 `OnlineUserHome.getService().logout(ctx, tokenId)`
     （同一個 DAO 查詢），不是 `TokenService.discard()`。兩個「token」子系統格式都是
     UUID、語意完全不同，是這次最容易踩錯的一個坑。
  4. **`logout()` 內部會跑既有的聊天在場狀態監聽器（`onlineUserRemoveListeners`），
     這個部署對非瀏覽器登入的 session 會在這一步丟 NPE**（`"sc" is null`，跟
     `ChatRoomHome.getService() is null` 是同一批既有、非本次改動造成的雜訊，
     `scripts/deploy-servlet.sh` 本來就會過濾同類噪音）——**token 本身在丟例外之前就已經
     刪除、真的失效了**，`LocalApiAuthHandler.discard()` 已經包一層
     try/catch+複查（`getItem(tokenId)` 確認真的沒了才重新拋出），不讓這個無關
     監聽器的例外蓋掉「其實撤銷成功」的結果。

  **DB 註冊（僅需一次，兩筆）**：
  ```sql
  INSERT INTO TsLocalApiGroup (FId, FName, FCode, FEnabled, FAuthType, FRequestFormat, FResponseFormat, FIsStandardResponse, FHandlerClassName, FDescription)
  VALUES ('a6f3b6c1-6e1a-4b8a-9b0e-2f1a7c9d4e10', '資產管理本地API', 'Ecp.Asset.LocalApi', b'1', 'Token', 'JSON', 'JSON', b'0', NULL, '...');

  INSERT INTO TsLocalApi (FId, FPath, FLocalApiGroupId, FEnabled, FTargetType, FTarget, FHandleType, FEnableRrConfig, FRequestFormat, FResponseFormat, FIsStandardResponse, FDescription)
  VALUES ('b7f4c7d2-7f2b-4c9b-8c1f-3f2b8d0e5f21', 'asset/api', 'a6f3b6c1-...', b'1', 'ApiMethod',
    'com.chainsea.ecp.asset.integration.AssetLocalApiHandler.process', 'Api', b'0', 'JSON', 'JSON', b'0', '...'),
  -- 第二筆 auth/token -> LocalApiAuthHandler.process，同一個 Group
  ```
  `FTargetType`/`FHandleType` 的字面值本身沒有語意（反編譯 `LocalApiModel.isActionMethod()`/
  `isFullHandle()` 確認只各自比對一個常數 `"ActionMethod"`/`"Full"`，只要不等於這兩個
  值，行為都一樣），選 `ApiMethod`/`Api` 純粹是取名字直覺、不是框架要求的固定枚舉。
  `FTarget` 格式是 `完整類別名.process`（因為走 `LocalApiProcessor` 介面而非
  `execute(req,resp)` 靜態方法那條路——反編譯 `DefaultLocalApiProcessor.process()`
  確認底層用 `ReflectUtil.invoke(target,...)` 對這個字串做 `lastIndexOf(".")` 拆
  類別+方法名，非 static 方法會自動 `newInstance()` 再呼叫，兩種 handler 寫法都吃
  同一個反射邏輯）。

  **部署方式**：跟這個 skill 目錄下所有手寫 servlet/class 一樣，用
  `bash scripts/deploy-servlet.sh asset-integration/src/com/chainsea/ecp/asset/integration/AssetLocalApiHandler.java asset-integration/src/com/chainsea/ecp/asset/integration/LocalApiAuthHandler.java`
  即可（含編譯+`docker cp`進`WEB-INF/classes`+檢查無開放會話後自動 restart）。

  **端到端測試序列（已完整跑過一輪，全部符合預期）**：
  ```bash
  # 1. 用服務帳號（agent/1234，只有 Ecp.Asset 的新增/查看/修改權限，沒有刪除）換 token
  curl -s -X POST ".../aipower/openapi/auth/token?op=apply" -H "Content-Type: application/json" \
    -d '{"loginName":"agent","password":"1234"}'
  # -> {"tokenId":"...","accountId":"...","accountName":"資產自動回報代理"}

  TOKEN=<上面拿到的 tokenId>

  # 2. list/create/update 全部成功
  curl -s -X POST ".../openapi/asset/api?op=list"   -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"pageSize":5,"pageIndex":1}'
  curl -s -X POST ".../openapi/asset/api?op=create" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"data":{"FName":"...","FAssetCode":"...","FCategory":"...","FStatus":"InUse"}}'
  curl -s -X POST ".../openapi/asset/api?op=update" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"id":"<id>","data":{...}}'

  # 3. delete 被真正的角色權限擋下（不是本機碼寫死擋，是走 ServiceContext.create(identity,...)
  #    觸發框架自己的 TsRolePrivilege 檢查）
  curl -s -X POST ".../openapi/asset/api?op=delete" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"id":"<id>"}'
  # -> {"errorCode":"Qs.Privilege.Check2","errorMessage":"您沒有\"資產管理\"的\"刪除\"許可權。"}

  # 4. 沒帶 token 直接被擋
  curl -s -X POST ".../openapi/asset/api?op=list" -H "Content-Type: application/json" -d '{}'
  # -> {"errorCode":"Asset.Token.Required"}

  # 5. 撤銷後舊 token 立即失效
  curl -s -X POST ".../openapi/auth/token?op=discard" -H "Authorization: Bearer $TOKEN"
  curl -s -X POST ".../openapi/asset/api?op=list" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{}'
  # -> {"errorCode":"Asset.Token.Invalid"}
  ```

  **這個結果代表使用者最初的需求（「我要用這個 api 來認證，跟輸入帳密是二選一的
  選項」）已經真的達成**：Flutter client 只在使用者第一次設定時打一次
  `auth/token?op=apply`（明文密碼直接送、不需要 RSA，因為 `isApiLogin=true` 且
  `keyId=null` 時框架自己不做解密），拿到 tokenId 後密碼就可以整個丟掉，之後所有
  背景回報都只用 `Authorization: Bearer <tokenId>`，且這個 tokenId 是可以隨時被
  這個帳號自己（或後台踢下線）撤銷的，不等於真實登入密碼——跟原本 DPAPI
  加密存密碼的方案比，多了「可撤銷、不落地密碼」這一層，且已經是目前這個部署裡
  唯一驗證過真的能動的做法。**Flutter 端要不要接這個新端點是下一步、目前尚未做**
  （目前 Flutter 仍在用 DPAPI 加密存密碼那版），純粹是使用者還沒明確要求切換前
  先不動既有已驗證穩定的路徑。

- **2026-07-23（再續）** 🚧 **進行中：讓 Flutter agent 像 SSH server 一樣即時待命，
  接收「主控台」下的任意 shell 命令**——使用者的新需求，跟前面 Token 認證是同一個
  Flutter 專案的延伸。目前後端（Java）已經寫完、部署成功、Jocket 握手/WebSocket
  連線本身確認可行，但「主控台推命令 → agent 收到並回結果」這條路徑實測還沒打通
  （見下面「目前卡住的地方」），**Dart/Flutter 端完全還沒寫**。

  **設計決策過程（重要，避免下次又繞一次）**：
  1. 使用者一開始要「任意 shell 命令 + 像 SSH server 一樣即時待命」，我一開始想的是
     「agent 自己在本機開一個 HTTP server 讓主控台連進來」——**使用者選了共享密鑰
     認證**，但這個方向後來被自己推翻了：辦公室電腦通常收不到外部連進來的連線
     （NAT/防火牆），要 agent 自己開對外 port 在多台機器規模下是真實的維運負擔。
  2. **改成 agent 主動連出去**（跟 aipower 保持一條長連線，主控台的命令透過這條連線
     轉推下來）——**這裡我犯了一個要記住的錯**：一開始跟使用者說「這個專案已經有
     Jocket 的現成範例可以直接抄」，查證後發現是講錯了——LINE 客服主控台那個既有
     管理台其實是**純輪詢**（`setInterval`），根本沒用 WebSocket。Jocket
     （`com.jeedsoft.jocket`）本身確實是框架內建、框架自己也真的在用的基礎設施
     （`OnlineUserServiceImpl` 踢除 session 時用 `InnerJocket.sendMessage()`
     推播），但這個專案裡**沒有自己寫過的、拿 Jocket 做雙向自訂命令通道的範例可抄
     ——一定要老實反編譯挖清楚**。**教訓**：講「這裡有現成範例」這種具體技術宣稱之前，
     要先真的查過，不要用「印象中應該有」帶過。
  3. 使用者明確要求「照樣用 Jocket，挖到底」，於是完整反編譯
     `jeedsoft-jocket-2.2.1.jar`（框架內建 jar，`WEB-INF/lib/`），下面完整記錄協定。

  **✅ Jocket 完整協定（反編譯確認，之後任何要接 Jocket 的需求直接用這份，不用再挖）**：

  1. **握手**：`POST /aipower/jocket/create`，query string 帶
     `jocket_path=/<你的 endpoint 路徑>`（**⚠ 一定要帶開頭斜線**——反編譯
     `JocketDeployer.addToTree()`/`getConfig()` 都用 `split(path,"/")` 且**跳過
     index 0 那個空字串**，這代表已註冊路徑的比對格式是「開頭斜線切出來的空字串+
     實際路徑」，`jocket_path` 沒帶開頭斜線會直接切法對不起來、回 404
     "Jocket configuration not found."——這個坑已經真的踩過一次）+ 任意你自訂的
     其他參數（反編譯 `JocketCreateServlet.doPost()` 確認：所有**不是**
     `jocket_` 開頭的參數，會原樣存進 `JocketSession.parameters`，可以用這個夾帶
     認證用的 tokenId、或任何你的 app 需要的識別資訊，`JocketEndpoint.onOpen()`
     裡用 `session.getParameter("xxx")` 讀回來）。回應 JSON：
     `{"sessionId":"...","pingInterval":25000,"pingTimeout":20000,"upgrade":true,"pathDepth":N}`。
     **關鍵**：`onOpen(session, httpSession)` 是在**這個 HTTP handshake 當下**
     同步呼叫的（`JocketEndpointRunner.doOpen()`），不是等 WebSocket 真的連上才觸發。
  2. **WebSocket 連線**：`ws://host/aipower/jocket/ws?s=<sessionId>`（query 參數
     名稱固定是 `s`，反編譯 `JocketWebSocketEndpoint.onOpen()` 確認）。連上後**必須
     先送一次** `{"type":"upgrade"}`（不送的話連線停在 probing 狀態，
     `JocketConnectionManager` 不會把它轉正，這個坑還沒實測到确切後果但反編譯邏輯
     很明確要送）。
  3. **心跳**：client 端主動、定期（每 `pingInterval` 毫秒）送
     `{"type":"ping"}`，server 回 `{"type":"pong"}`（`JocketWebSocketEndpoint.
     onMessage()` 收到 `type=="ping"` 才回 pong，是被動的，client 不送就沒有
     心跳）。
  4. **App 訊息（雙向）**，同一個扁平 JSON envelope（`JocketPacket.toJson()`/
     `parse()` 反編譯確認欄位）：`{"type":"message","name":"<event>",
     "data":"<data 的 JSON 字串，注意是字串不是巢狀物件，要 parse 兩次>",
     "id":"<可省略>"}`。
     - **Server → client 推播**：Java 端呼叫 `com.jeedsoft.jocket.Jocket.send(
       sessionId, event, dataObject)`（靜態方法，內部就是組上面這個 envelope
       再 `JocketQueueManager.publish()`）。
     - **Client → server**：送同樣格式的 frame，framework 呼叫你的
       `JocketEndpoint.onMessage(session, name, data)`（`data` 一樣是字串，
       自己 `new JSONObject(data)` 解）。
  5. **endpoint 部署方式**：寫一個 class `implements
     com.jeedsoft.jocket.endpoint.JocketEndpoint`（`onOpen`/`onClose`/
     `onMessage` 三個方法），標
     `@com.jeedsoft.jocket.endpoint.JocketServerEndpoint("/你的路徑")`，在某個
     `ServletContextListener.contextInitialized()` 裡呼叫
     `JocketDeployer.deploy(YourEndpoint.class)`（`throws JocketException`，
     要包 try/catch）。**不用自己呼叫 `JocketService.start()`**——這個部署本身
     Jocket 服務早就被框架自己啟動了（`JocketHome.initialize()`，開機 log 會印
     `[Jocket] Service started. tree structure: |---jocket |---inner
     |---mobile`），只需要 `deploy()` 掛上你自己的新路徑，成功後開機 log 的樹狀圖
     會多一行 `|---<你的路徑> (你的 class)`——**這是驗證部署成功最快的方法，比
     猜測有沒有生效準**。

  **✅ 已完成、已編譯部署成功的程式碼**（`asset-integration/src/com/chainsea/ecp/
  asset/integration/`）：
  - `LocalApiAuth.java`：把 `AssetLocalApiHandler` 原本內嵌的 Bearer token 解析
    邏輯抽成共用 class（`resolveIdentityFromBearerToken()`），`AssetLocalApiHandler`
    也改用這個。
  - `AgentControlEndpoint.java`：`@JocketServerEndpoint("/agent-control")`。
    `onOpen()` 讀 handshake 帶的 `tokenId`/`assetCode` 參數、驗證 tokenId（跟
    `LocalApiAuth` 同一套 `OnlineUserHome.getService().getItem(tokenId)`），驗證
    通過就把 `assetCode -> jocket sessionId` 存進一個行程內 `ConcurrentHashMap`
    （`ONLINE_AGENTS`，純記憶體、無落地、無叢集支援，container 重啟全部重來，
    這是刻意先做的最小可行版本）。`onMessage()` 處理 agent 回傳的
    `name=="result"` 訊息，用 `requestId` 對應一個等待中的
    `CompletableFuture<JSONObject>`（`PENDING` map）並 complete 它。
  - `AgentControlLocalApiHandler.java`：新的本地 API（`FPath=agent/control`，
    `op=execute`），驗證呼叫端（「主控台」操作者）的 Bearer token（**目前只驗證
    「是不是登入過的 aipower 帳號」，沒有「誰能對哪些機器下命令」這層授權
    ——刻意先做最小可行版本，正式對外開放前要補**），從 `body.assetCode` 找
    `ONLINE_AGENTS` 裡的 sessionId（找不到回 `Agent.Offline`），生成
    `requestId`、建一個 `CompletableFuture` 放進 `PENDING`、
    `Jocket.send(sessionId,"execute",{"requestId":...,"command":...})`
    推下去，然後 `future.get(timeoutSeconds, SECONDS)` 卡住這次 HTTP 呼叫等
    agent 把結果推回來（逾時預設 30 秒、上限 120 秒）。
  - `LineIntegrationStartupListener.java`（`line-integration/` 目錄下，共用既有
    StartupListener 掛新功能是這個 skill 已經確立的慣例，見上面 LINE 主控台那節）
    加了 `JocketDeployer.deploy(AgentControlEndpoint.class)`。
  - DB 註冊：`TsLocalApi` 新增一筆 `FPath=agent/control`（沿用跟
    `asset/api`/`auth/token` 同一個 `TsLocalApiGroup`，`FRequestFormat`/
    `FResponseFormat` 只有 Group 那筆的值真的生效，因為
    `FEnableRrConfig=0`——每筆自己的 FRequestFormat/FResponseFormat 是死欄位，
    這個坑上一節已經記過一次）。
  - **部署已驗證**：開機 log 確認 `|---agent-control
    (com.chainsea.ecp.asset.integration.AgentControlEndpoint)` 有掛上。

  **⚠ 目前卡住的地方（下次接續從這裡查）**：寫了一支 Python 測試腳本
  （`websockets`+`requests`，同時模擬「agent 端」跟「主控台端」）驗證完整流程：
  1. `auth/token?op=apply` 拿 tokenId ✅
  2. `jocket/create?jocket_path=/agent-control&tokenId=...&assetCode=TEST-AGENT-001`
     拿到 sessionId ✅（一開始漏了開頭斜線踩了上面記錄的那個坑，修好後 200 OK）
  3. WebSocket 連上 `jocket/ws?s=<sessionId>` ✅（有連上，log 印
     `[agent] ws connected`）
  4. 呼叫 `agent/control?op=execute` 觸發 `Jocket.send(sessionId,"execute",...)`
     ❌ **10 秒逾時，agent 端的 WebSocket 完全沒收到任何 `execute` 訊息**
     （沒印出 `[agent] recv: ...`）。

     還沒查出根因，下次接續的排查方向：
     - 反編譯確認 `onOpen()` 是在 `/jocket/create` **HTTP handshake 當下**同步
       執行、拿到的 `session.getId()`（我存進 `ONLINE_AGENTS` 的那個 ID）跟後續
       WebSocket 連線用的 `?s=<sessionId>` 理論上是同一個 ID——但沒有實際加
       log 驗證這個假設，第一步應該先加 debug log 印出 `onOpen()` 裡拿到的
       session id、`Jocket.send()` 呼叫時查到的 session id，兩邊比對是否真的一致。
     - 懷疑「WebSocket 建立連線」（`JocketWebSocketEndpoint.onOpen`，反編譯過但
       沒細看）跟「HTTP handshake 建立的 JocketSession」之間可能有第二層
       綁定/替換邏輯沒摸清楚——`JocketConnectionManager.addProbing()` +
       `type=upgrade` 那段「probing -> 正式連線」的轉換，有沒有可能連線物件
       本身在轉正過程換了一個新的內部識別碼，導致 `Jocket.send(舊 sessionId,...)`
       實際上找不到轉正後的連線？這是最可疑的方向，還沒反編譯
       `JocketQueueManager.publish()`/`JocketConnectionManager` 內部怎麼把
       session id 對應到實際連線物件。
     - 已經開了 `log4j2.xml` 的 `com.jeedsoft.jocket` trace logger 的編輯（存在
       本機暫存 `C:\Users\HCH\AppData\Local\Temp\jocket-explore\log4j2.xml`，
       **還沒 docker cp 回去**）——下次先把這個 trace logger 真的打開、重跑一次
       測試腳本、看 trace log 完整還原 `Jocket.send()` 那一刻框架內部到底找不找
       得到這個 session/connection。
     - 測試腳本存在
       `C:\Users\HCH\AppData\Local\Temp\claude\...\scratchpad\jocket_test.py`
       （session 暫存目錄，可能會被系統清掉，之後要重寫一份存進這個 skill 目錄
       比較保險）。

  **完全還沒做**：Dart/Flutter 端的 Jocket client（連線/心跳/收發訊息/實際
  `Process.run` 執行 shell 命令並回傳結果）、UI 上顯示「遠端命令通道：已連線/
  離線」的狀態列、以及「主控台」本身（誰來呼叫 `agent/control?op=execute`）—
  目前只用 Python 測試腳本模擬過主控台端，沒有真正的 UI/管理頁面。

- **2026-07-23（再續2）** ✅ **上面卡住的 Jocket 謎團解開了：反編譯
  `jeedsoft-jocket-2.2.1.jar` 找到真正原因，純 WebSocket client 天生就打不通，
  一定要先做 HTTP long-polling bootstrap。** 用 CFR 把整個 jar 反編譯出來逐層追
  `Jocket.send()` 的呼叫鏈：
  ```
  Jocket.send(sid,name,data)
    → JocketQueueManager.publish(sid, packet)     // packet.type="message"
      → JocketLocalQueue.publish()
        → preparePublish(session, packet)          // 檢查 session.status=="open" 才放行
        → queue.add(packet)                        // 封包放進以 sid 為 key 的記憶體佇列
        → JocketConsumer.notify(sid)
          → JocketConnectionManager.get(sid)        // ⚠ 只查 connections map，不查 probingConnections！
          → 如果是 null：直接 return，什麼都不做（連 log 都只有 trace 等級，肉眼完全看不到)
  ```
  而 `JocketWebSocketEndpoint.onOpen()`（WebSocket 剛連上時）只會呼叫
  `JocketConnectionManager.addProbing(cn)`，把連線放進 `probingConnections`——
  **從 `probingConnections` 搬到 `connections`（也就是「真正升級完成」）的唯一
  程式碼路徑，藏在 `JocketPollingConnection.downstream()`**：
  ```java
  // JocketPollingConnection.downstream()
  if ("upgrade".equals(packet.getType()) && !JocketConnectionManager.upgrade(sessionId) ...)
  ```
  也就是說：client 端送出 WebSocket 的 `{"type":"upgrade"}` 之後，這個 upgrade
  封包一樣要先進佇列、一樣要靠 `JocketConsumer.notify()` 才會被處理——但
  `notify()` 一樣只查 `connections`，如果這時候 `connections` 裡什麼都沒有
  （純 WebSocket client 从沒建立過 polling 連線），一樣直接 no-op，**upgrade
  封包永遠沒人處理，WebSocket 永遠卡在 probing，`Jocket.send()` 之後所有訊息
  都进了一個沒人看的佇列，無限期沉默**。這是一個雞生蛋蛋生雞的死結：upgrade
  封包需要靠佇列消費者處理，但佇列消費者只認得已經在 `connections` 裡的連線，
  而唯一能把 WebSocket 連線放進 `connections` 的動作，剛好就是處理 upgrade
  封包本身要做的事。

  **正確協定（比對 `JocketPollingServlet`/`JocketCreateServlet` 反編譯結果，
  這其實就是標準 engine.io/socket.io 風格的「先 polling、後升級」握手，Jocket
  這個框架顯然是照抄同一套設計）**：
  1. `POST /jocket/create?jocket_path=/xxx&...` 拿 `sessionId`（`session.status`
     這時已經是 `"open"`，但完全沒有任何 connection 物件存在）。
  2. **★ 缺這步就永遠打不通**：立刻發一個 `POST /jocket/poll?s=<sessionId>`
     （伺服器端用 Servlet `AsyncContext` 掛起、不會馬上回應）——這會建立一個
     `JocketPollingConnection` 並呼叫 `JocketConnectionManager.add()`，這是
     整個流程裡**唯一**會讓某個 connection 出現在 `connections` map 的地方。
  3. 開 `ws://.../jocket/ws?s=<sessionId>`，`onOpen()` 把它放進
     `probingConnections`。
  4. WebSocket 送 `{"type":"upgrade"}` → 進佇列 → `notify()` 這次終於在
     `connections` 找到步驟 2 那個 polling 連線 → 呼叫它的
     `onEvent()`→`downstream()` → 把 upgrade 封包透過那個掛起的 HTTP 回應送
     回 client（原本 pending 的 `/jocket/poll` 這時候才會真的回應）、**同時**
     呼叫 `JocketConnectionManager.upgrade(sessionId)`，把 WebSocket 連線從
     `probingConnections` 搬進 `connections`，取代掉 polling 連線。
  5. 從此之後 `connections.get(sessionId)` 回傳的才是 WebSocket 連線，後續
     `Jocket.send()` 才能真正送達。

  **實測驗證**：用 Node.js（v24 內建 `fetch`/`WebSocket`，不用裝套件）寫了
  補上 polling bootstrap 那一步的新測試腳本
  （`jocket-protocol-notes/jocket_handshake_test.mjs`，用法
  `node jocket_handshake_test.mjs <有效的 aipower session tokenId>`），跑完
  清楚看到 `[poll got] {type:'upgrade'}` 接著 `[ws recv]` 收到後端
  `AgentControlLocalApiHandler` 送出的 `execute` 訊息——**這正是上次卡住整整
  一輪、反覆懷疑 session id 對應關係的地方，現在確認純粹是「少做 polling
  bootstrap」，不是 session id 或 Registry 哪裡對不上**。

  **下一步（還沒做）**：`AgentControlEndpoint`（Java 後端）跟未來的 Flutter
  Dart client 都要照這個五步驟實作 handshake，特別是 Flutter 端一定要先送一個
  `/jocket/poll` 請求（可以送完立刻等它的 response 而不用真的處理內容，純粹
  是為了讓 polling connection 進到 `connections` map 觸發升級）才能開始收
  WebSocket 推播；也可以研究 Jocket 官方是否有現成的 JS/其他語言 client 函式庫
  把這個握手细节包好，直接用會比手刻可靠。Dart/Flutter 端實作本身仍完全還沒動工。

- **2026-07-23（再續3）** ✅ **Flutter 端 Jocket client 寫完、端到端驗證通過
  ——「從 aipower 送命令給 Flutter agent 執行、結果送回 aipower」整個功能正式
  可用。** 專案位置 `C:\Users\HCH\flutter_tray_app`（跟這個 skill 資料夾分開，
  是獨立的 git 專案，不需要複製回這裡）。新增兩個檔案：
  - `lib/services/jocket_client.dart`：通用 Jocket protocol client，照上一節
    反編譯出來的協定實作完整握手（create→發 `/jocket/poll` bootstrap（不
    await，背景跑）→ 等 400ms → 開 WebSocket → 送 `{"type":"upgrade"}` →
    等 poll 那個 Future 完成才算真正連上）；`messages` stream 給外部收
    `{name,data}`、`send(name,data)` 送出去，斷線會透過 `onDisconnect`
    stream 通知外層。用 `web_socket_channel: ^3.0.1`（新加的 pubspec 依賴，
    Dio 已經在用不用重複加）。
  - `lib/services/agent_control_service.dart`：接上 `/agent-control`，收到
    `execute` 訊息（`{requestId,command}`）就 `Process.run('cmd.exe',
    ['/c', command])` 執行、把 `{requestId,exitCode,stdout,stderr}` 用
    `result` 訊息送回去；斷線 10 秒後自動重連；密碼模式的憑證會先呼叫既有
    `AssetLocalApiClient.applyToken()` 換一個 token 專門給這條長連線用
    （跟 `AutoReportService` 自己的 token 互不影響，各自可獨立撤銷）。
  - `main.dart`/`asset_list_screen.dart`：跟 `AutoReportService` 一起在
    `_setupAutoReport()` 裡啟動，UI 加了「遠端命令通道：已連線/離線」狀態列
    （樣式照抄既有的「自動回報」狀態列）。

  **⚠ 實測抓到一個大小寫坑，值得記住**：一開始 `assetCode` 用
  `Platform.localHostname` 組（`'AUTO-${Platform.localHostname}'`），UI
  顯示「已連線」（Jocket 連線本身沒問題），但主控台送命令回
  `Agent.Offline`——查出來這台機器 `Platform.localHostname` 回傳
  `Gram`（跟 Git Bash `hostname` 指令一樣的原始大小寫），但
  `AutoReportService`／`SystemInfoService.detect()`（PowerShell
  `$cs.Name`）查到、DB 裡實際存的是全大寫 `GRAM`。後端
  `AgentControlEndpoint.ONLINE_AGENTS` 是普通 `Map<String,String>`，key
  比對大小寫敏感，兩邊 assetCode 字串對不上就等於「查無此 agent」——**症狀
  是 Jocket 連線正常（onOpen 驗證 token 通過就會顯示已連線）但主控台永遠
  找不到這台機器**，容易誤判成 Jocket 協定又有問題，其實是完全不同層次
  的 bug。修法：`AgentControlService` 改成呼叫跟 `AutoReportService` 完全
  相同的 `SystemInfoService.detect()` 取得 hostname（快取一次，不用每次
  重連都重跑 PowerShell），保證兩邊 assetCode 字串來源一致。**教訓：任何
  跨模組共用的識別字串（這裡是 assetCode），一定要共用同一個產生邏輯/同
  一次查詢結果，不能各自用「看起來應該一樣」的不同 API 各查一次**——像
  `Platform.localHostname` 跟 PowerShell CIM 查到的電腦名稱，字面上都叫
  「hostname」，實際字串可能不同。

  **端到端驗證**：`flutter analyze` 乾淨、`flutter build windows --debug`
  編譯成功；真人在 Windows 機器上重開 app 確認「已連線」後，從 host 端
  curl 打 `POST /openapi/agent/control?op=execute`
  `{"assetCode":"AUTO-GRAM","command":"echo hello-from-aipower-console &&
  hostname"}`，拿到
  `{"stdout":"hello-from-aipower-console \r\nGram\r\n","exitCode":0,
  "stderr":""}`——命令真的在那台機器上執行、結果正確傳回來，完整鏈路
  （aipower console → Jocket → Flutter → cmd.exe → 回傳）全部打通。

  **還沒做**：真正的「主控台」UI（目前呼叫端只用 curl 手動測試，沒有網頁
  介面可以選機器、輸入命令、看結果）；`agent/control` 呼叫端目前沒有
  「誰能對哪些機器下命令」的授權層，只驗證「是不是登入過的 aipower 帳號」
  （`AgentControlLocalApiHandler.java` 註解裡就寫明是刻意的最小可行版本，
  正式對外開放前要補）。

- **2026-07-23（再續4）** ✅ **「主控台」網頁介面補上了——資產遠端命令功能
  完整收尾（選機器→輸入命令→看結果，都在瀏覽器上做）。** 新增
  `AgentConsoleServlet`（`com.chainsea.ecp.asset.integration` package，跟
  `AgentControlEndpoint` 同目錄），掛在 `/aipower/asset/agent-console`：
  - **重構**：把 `AgentControlLocalApiHandler`（Bearer token 呼叫端，給
    Flutter/外部程式用）裡「送命令、等結果」那段 `PENDING`/
    `CompletableFuture`/`Jocket.send()` 邏輯抽成
    `AgentControlEndpoint.execute(assetCode,command,timeoutSeconds)`
    静态方法，加上 `AgentControlEndpoint.listOnlineAgents()`（回傳目前
    `ONLINE_AGENTS` 的排序清單）——兩個呼叫端（Bearer token／ECP session
    cookie）共用同一份核心邏輯，不重複維護。`AgentControlLocalApiHandler`
    瘦身成只剩「驗證 Bearer token → 解析 body → 呼叫共用方法」。
  - `AgentConsoleServlet` 走 ECP session 驗證（`LineIntegrationHome.
    getCurrentOnlineUser(req)`，跟 `AcdAdminServlet`/`ChatReportServlet`
    同一套），三個路由：`GET /agent-console`（頁面）、
    `GET /agent-console/agents`（目前在線 agent 清單，5 秒輪詢）、
    `POST /agent-console/execute`（`{assetCode,command,timeoutSeconds}`，
    呼叫 `AgentControlEndpoint.execute()`，`SystemException` 的
    `Agent.Offline`→409、`Agent.Timeout`→504、其他→500）。頁面本身是純
    HTML/CSS/JS（跟這個 skill 其他自建頁面同一套風格：下拉選機器＋文字框
    輸入命令＋逾時秒數＋送出按鈕，執行紀錄由新到舊疊在下面，每筆顯示
    exitCode/stdout/stderr 或錯誤訊息），沒有外部函式庫依賴。
  - **`LineIntegrationStartupListener`** 加了第 7 個 `addServlet` 區塊註冊
    這支 servlet（`setLoadOnStartup(7)`），沿用「借用既有 StartupListener，
    不改 web.xml」的既定慣例。
  - **TsMenu**：新增一筆「資產遠端命令主控台」（`FType='ExternalPage'`，
    `FExternalPageUrl='/aipower/asset/agent-console'`，同源相對路徑——
    沿用 2026-07-20 那次 LINE 聊天框 iframe session cookie 沒帶到的教訓），
    掛在「開發→工具」底下 `004.001.010`（跟 MikoPBX/LINE/Asset/AiProxy
    那幾個工具同一層、同樣沒有對非管理員角色開 `TsRoleMenu`——這是刻意的：
    這支 servlet 目前完全沒有授權層，只驗證「登入過」就能對任何在線機器
    下任意 shell 命令，先維持只有 Administrator 看得到選單這層防線）。
  - **部署**：4 個檔案（`AgentControlEndpoint`/`AgentControlLocalApiHandler`
    /`AgentConsoleServlet`/`LineIntegrationStartupListener`）一次用
    `scripts/deploy-servlet.sh` 編譯部署（`open_rooms=0` 確認過），
    `docker restart aipower-app` 後 log 無新錯誤；TsMenu 是純 SQL、不用
    重啟，改後重新登入生效。
  - **驗證**：`curl` 測三個端點——頁面 200、`agents`/`execute`（未帶
    session）皆正確回 401，符合預期的登入檢查。**尚未走完整瀏覽器操作**
    （debug Chrome 的 CDP（當時的舊独立設定）這次連不上，見這個 skill「已知踩坑」章節
    同樣的狀況），只驗證到後端邏輯跟 HTTP 層——選機器下拉、送出命令、看
    執行結果這一段 UI 互動流程之後應該找時間用 `/connect-chrome`
    重新接管後補測。核心「送命令拿結果」邏輯本身已經在上一輪透過
    Bearer token 路徑端到端驗證過（真的在 Flutter 那台機器上執行
    `echo`+`hostname` 拿到正確結果），這次只是換一個新的呼叫端（網頁）
    共用同一份邏輯，風險相對低。

- **2026-07-23（再續）** ✅ **補齊 `Ecp.AiProxy`/`Ecp.LineIntegration` 兩個「半套」Unit**。
  背景：比對乾淨環境跟已加服務的 dump 時發現這兩個是例外——`Ecp.AiProxy` 雖然是真的
  TsUnit，但 `FModuleId`/`FEditId`/`FIcon`/`FOpenMode`/`FDataSource`/`FKeyField`/
  `FKeyType`/`FNameField` 全 NULL，且 `FDaoClassName`/`FServiceClassName` 指向**介面**
  不是實作類別；`lineintegration` 則完全沒有 TsUnit（原生 servlet 架構，見上面「LINE
  客服主控台」章節，這個不算 bug，是刻意設計）。使用者要求「補齊坑，改成標準寫法」。
  - **反編譯（CFR，而非只看 `javap -p` 簽章）確認 AiProxy 的 Dao/Service/Action 三層
    其實都已經正確寫了 `super(UNIT_ID, Model.class)` 建構子**——一開始以為會踩
    「沒寫建構子」那個致命坑（`javap -p` 只印簽章不印建構子內容，容易誤判成空殼），
    反編譯後證實三層都沒問題，省下重寫六大核心的功夫。**教訓：判斷「建構子有沒有正確
    呼叫 super」一定要反編譯看方法體，不能只看 `javap -p` 的簽章列表**。
  - **真正的半套根因**：`AiProxyStartupListener.ensureSchema()` 每次容器啟動都跑一次
    `INSERT IGNORE INTO TsUnit (FId,FCode,FName,FTable,FActionClassName,FHomeClassName,
    FDaoClassName,FServiceClassName)`（只給 8 個欄位、Dao/Service 填介面），因為
    `INSERT IGNORE` 對已存在的 FId 是 no-op，手動 SQL 修正後不會被蓋回去，但**原始碼
    不修的話下次全新建置/換資料卷還是會重新產生半套版本**。已重寫
    `AiProxyStartupListener.java`（拿掉這段自建 TsUnit 的邏輯，只保留
    `CREATE TABLE IF NOT EXISTS TcAiProxySetting`），編譯部署（`scripts/deploy-servlet.sh
    --no-restart`，因為要跟後面的 SQL 一起套用再統一重啟一次）。
  - **AiProxy 標準化 SQL**（`aiproxy-integration/aiproxy_standardize.sql`）：`UPDATE
    TsUnit` 補齊全部標準欄位（照 `Ecp.MikopbxSetting` 的模板）+ 完整 13 張表（`TsField`
    ×5、`TsList`/`TsListField`、`TsPage`(List+Form)、`TsForm`/`TsFieldGroup`/
    `TsFormField`、`TsMenu`(掛「開發→工具」`004.001.009`，接續 `Ecp.Asset` 的
    `008`)、`TsToolItem`×5、`TsPrivilege`×4 + `TsRolePrivilege`×12（照
    `Ecp.MikopbxSetting` 模板：租戶管理員/員工角色/預設角色三個角色各 4 筆，`FGlobal`
    皆 `NULL`，這是內建角色的既有慣例，跟後面「授權給自訂角色」那次刻意設 `b'1'`
    是兩回事）。**安全設計**：`TcAiProxySetting.FValue` 明碼存 DeepSeek API Key／DB
    密碼／MikoPBX 密碼，`TsListField` 刻意不放 `FValue`（清單只顯示 Key/說明/更新
    時間，不會一開清單就整批明碼曝光）；且「開發→工具」這整個目錄本來就沒有對任何
    非管理員角色開 `TsRoleMenu`（查證過，0 筆授權），所以新清單頁的實際曝險範圍跟
    現有的 8 個兄弟 Unit 完全一致，只有 Administrator（本來就能用「執行SQL」看到一切）
    看得到。
  - **LineIntegration 裝飾性 Unit SQL**（`line-integration/lineintegration_decorative_unit.sql`）：
    只有 `TsUnit`（class-name 四欄全 NULL）+ 3 個 `TsField`，**刻意不建**
    `TsList`/`TsPage`/`TsMenu`/`TsToolItem`/`TsPrivilege`——純粹讓它在「單元列表」查
    得到、看起來跟其他模組一致，客服主控台本身完全不受影響、繼續是
    `/aipower/line/console` 原生 servlet。新建一張極簡佔位表 `TcLineIntegrationInfo`
    （單筆靜態說明），刻意不指向 `TcChatRoom`/`TcChatMessage`，避免跟正在被 servlet
    直接讀寫的原生表產生任何交集風險。
  - **部署順序**（照這個 skill 一貫的安全規範，檢查跟重啟分開跑，不包在同一個指令裡）：
    確認 `open_rooms=0` → 套用兩份 SQL → **再次獨立確認** `open_rooms=0`（時間有經過，
    重新查一次）→ `docker restart aipower-app` → 等 curl 200 → 查 log 排除已知雜訊後
    確認無新錯誤 → 查 DB 確認兩筆 `TsUnit` 都在且欄位正確。全部通過，`/health` 端點
    正常。**尚未做的**：沒有走瀏覽器實際開 `Ecp.AiProxy` 新清單頁點過新增/編輯/保存
    （只驗證到 DB metadata 正確、Registry 沒有啟動期錯誤——按 2026-07-22 那次教訓，
    這個框架版本 Unit 是 lazy 註冊，真正可不可以用要等第一次被呼叫時才知道，之後有
    機會應該補一次真實瀏覽器操作或直接呼叫 CRUD 端點驗證）。
  - **附帶修正**：使用者反映 Flutter 資產管理 App 裡刪除按鈕點了沒反應，查證是
    Flutter 用的服務帳號「agent」（角色「資產自動回報代理」）2026-07-23 建立時就
    刻意只給新增/查看/修改，沒給刪除（背景自動回報帳號的安全考量）。使用者確認要
    直接開放，補了一筆 `TsRolePrivilege`（`FGlobal=b'1'`，跟這個角色既有三筆的模式
    一致）授予刪除權限，`TsPrivilege`/`TsRolePrivilege` 是動態讀取即時生效的表
    （不像 `TsUnit`/`TsField`/`TsToolItem` 需要重啟），DB 層面已確認四個動作全部
    授權到位，實際 Flutter 端刪除是否成功留待使用者下次操作時確認。

- **2026-07-23（再續5）** ✅ **「主控台」不再是獨立選單項目，改成整個「資產管理」
  換成自訂頁面，清單＋遠端命令合併在一起。** 使用者反饋：不要另外建一個「主控台」
  頁面，希望直接在「資產管理」裡選要下命令的機器。

  新增 `AssetConsoleServlet`（取代上一節的 `AgentConsoleServlet`，該檔已刪除、
  註冊也拿掉），掛在 `/aipower/asset/console`，把原本標準 `Ecp.Asset.List`
  （`EntityList.jsp`）整頁換掉：
  - `TsMenu`「資產管理」那筆從 `FType='InternalPage'`+`FPageId`（指向標準清單頁）
    改成 `FType='ExternalPage'`+`FExternalPageUrl='/aipower/asset/console'`
    （同源相對路徑）。**`Ecp.Asset` 這個 TsUnit、`Ecp.Asset.List`/`.Form` 那兩張
    TsPage 本身都沒有刪除**，只是選單不再指過去——`Flutter` 端走的
    `/openapi/asset/api`（`AssetLocalApiHandler`，Bearer token）完全不透過這兩張
    TsPage/EntityList 渲染，兩條路徑各自獨立、互不影響，這也是敢放心把瀏覽器端
    整頁換掉的原因。
  - **CRUD 改用原生 JDBC 直接讀寫 `TcAsset`**（跟 `AcdAdminServlet` 同一套既有
    慣例），不透過 `EntityService`/`ServiceContext`——那一層是給標準 Action 呼叫鏈
    用的，從原生 Servlet 手動組一個正確的 ServiceContext 成本高、這幾個姊妹
    servlet 至今都選擇繞開它直接下 SQL。**技術欄位**（CPU/RAM/硬碟/OS/主機名稱/
    IP/最後回報時間，Flutter 自動回報代理寫入的）跟**保管人**（JOIN
    `TsAccount` 顯示名稱）**只唯讀顯示，不提供編輯**——這些欄位的權威來源是
    Flutter 端自動偵測，人工編輯意義不大，故意縮小這次的範圍。可編輯欄位：
    名稱/資產編號/分類/狀態/位置/序號/採購日期/採購金額/保固到期，涵蓋原本
    Flutter App 表單能編輯的範圍。
  - **UI 結構**：頂部搜尋+分類/狀態篩選（沿用 Flutter App 原本的篩選邏輯，
    純前端 filter 已載入的 JSON 陣列）+「新增資產」按鈕（展開一塊表單面板，
    編輯也共用同一塊）；清單每列：名稱/編號/分類/狀態/位置/**線上狀態圓點**+
    「詳情/命令」「編輯」「刪除」三個按鈕；點「詳情/命令」**展開一個 `detailRow`**
    顯示唯讀技術欄位九宮格＋（若該資產 assetCode 目前在線）一個命令輸入框+
    逾時秒數+送出按鈕+結果 `<pre>` 區塊——跟資產清單、詳情、送命令全部在同一頁，
    不用切換選單。
  - **部署**：`open_rooms=0` 確認過，`scripts/deploy-servlet.sh` 編譯部署
    `AssetConsoleServlet`+`LineIntegrationStartupListener`（後者要重啟才會重新
    註冊 servlet 對照表），`docker restart aipower-app` 後 log 無新錯誤；TsMenu
    的 UPDATE/DELETE 是純 SQL，改後重新登入生效（不用等重啟）。
  - **驗證**：`curl` 確認頁面 200、`assets`/`execute`（未帶 session）皆 401、
    舊的 `/asset/agent-console` 正確變成 404（確認整個取代掉了，沒有殘留兩條
    並存的路徑）。**這次 debug Chrome 的 CDP 又連不上**（連續第二次遇到，見上面
    同樣的已知狀況），沒能實際在瀏覽器裡點過搜尋/篩選/新增/編輯/刪除/展開詳情/
    送命令這整套互動——只驗證到 HTTP 層跟選單/TsUnit 資料完整性，之後應該找
    時間用 `/connect-chrome` 重新接管、走一次真人操作流程完整驗收（新增一筆
    測試資產、確認能編輯/刪除、找一台在線機器送一次真實命令）。

- **2026-07-23（再續6）** ✅ **`AssetConsoleServlet` 輪詢清空輸入框 bug + 「容器重啟會打斷
  已連線的 flutter_tray_app」連鎖問題，兩個都修好，衍生兩支可重用腳本。**

  **Bug 1：8 秒輪詢把使用者正在打的命令清空**——`PAGE_HTML` 內的
  `setInterval(loadAssets, 8000)` 每次都整個重建 `listBody` 的 DOM（含展開中的
  `detailRow`），導致使用者在命令輸入框（`cmd_<id>`）打到一半的字被蓋掉。修法：
  `loadAssets()` 在 `render()` 前後分別存/還原目前展開列的 `cmd_`/`to_` 輸入框
  內容＋游標位置（`selectionStart`），焦點也一併還原。`scripts/deploy-servlet.sh`
  編譯部署 `AssetConsoleServlet.java`，`docker restart aipower-app` 生效。

  **Bug 2（更關鍵，連鎖反應）：任何 `docker restart aipower-app` 都會讓所有已連線的
  flutter_tray_app agent 斷線，且不一定會自己好。** 根因：`AgentControlEndpoint`
  的 `ONLINE_AGENTS`/`PENDING`，以及框架 `OnlineUserStore`（Bearer tokenId 的
  有效性來源）都是**行程內記憶體**，JVM 重啟全部歸零。已連線的 tray app 會斷線，
  按設計會在 10 秒後自動重連（`AgentControlService._scheduleReconnect`）——**但
  這次実測發現兩層陷阱**：
  1. 桌面上已安裝的 `%LOCALAPPDATA%\Programs\AssetAgent\flutter_tray_app.exe`
     跟 `C:\Users\HCH\flutter_tray_app` 原始碼**版本沒對齊**（裝好的 build 比
     最新原始碼舊了將近一小時），單純重啟*行程*沒用——舊版本編譯進去的邏輯
     本身有問題，行程重開還是跑一樣的舊邏輯，UI 卡在
     `自動回報：...AipowerApiException: E.Asset.Token.Invalid` +
     `遠端命令通道：離線（連線中斷，準備重連）`永遠不會自己好。
  2. 診斷過程中發現：**光看 tray app 自己的狀態文字，判斷不出後端真實狀態**——
     UI 顯示離線的同時，用 Bearer token 直接呼叫
     `POST /openapi/agent/control?op=execute` 打同一個 assetCode 卻能成功執行
     命令（後端 `ONLINE_AGENTS` 其實有這個 assetCode）。不要只信任 tray app 的
     狀態列或網頁「線上/離線」圓點，先用下面的 `check-agent-status.sh` 對後端
     問一次「這個 assetCode 現在真的收得到命令嗎」，才知道問題出在後端連線本身、
     還是只是 UI 沒更新/裝的版本太舊。

  **真正的修法：重新編譯＋覆蓋安裝＋重啟，不是只重啟行程**——
  `flutter build windows --release` → `robocopy` 覆蓋
  `AppData\Local\Programs\AssetAgent` → 啟動新版 exe → 確認行程存活 30 秒
  （曾經觀察到剛啟動時活著、幾分鐘後行程卻消失，不確定是使用者手滑關掉視窗還是
  真的當掉，穩妥起見一律等 30 秒才算數）→ 用 `check-agent-status.sh` 打一次真實
  命令驗證。**已整支腳本化，之後同樣情境不要再手動一步步重新推導**：

  - `asset-integration/check-agent-status.sh <assetCode> [loginName] [password]`
    ——不開瀏覽器，直接用 Bearer token 對指定 assetCode 送一個
    `echo online-check` 測試命令，回傳真的在不在線（exit 0=在線／1=離線／
    2=其他錯誤）。預設帳密 `agent`/`1234`（資產自動回報代理帳號，密碼有輪替過
    要記得覆蓋參數）。
  - `asset-integration/rebuild-reinstall-tray-agent.sh [assetCode-to-verify]`
    ——包辦「重新編譯 → 殺舊行程 → robocopy 覆蓋安裝 → 啟動 → 等 30 秒確認活著
    → 呼叫 `check-agent-status.sh` 驗證」全流程，預設驗證 `AUTO-GRAM`（這台機器）。
    **`docker restart aipower-app` 之後如果使用者反映 flutter_tray_app 顯示離線
    / token 失效，直接跑這支，不要先猜測是不是行程卡住去單純重開行程**——上次
    就是先只重開行程結果沒用，繞了一圈才發現要整個重編譯安裝。

  **一個尚未解決、留給之後的謎團**：解密 `agent_credentials.bin`
  （DPAPI，見下面的解密片段）證實這台機器上的憑證檔其實是**密碼模式**（沒有
  `mode` 欄位，`loginName:"Agent"`／`password:"<TEST_PASSWORD>"`）——照 `credential_store.dart`
  的 `fromJson` 邏輯，密碼模式的 `AutoReportService`/`AgentControlService` 應該
  每次都重新登入拿新 session，理論上不該出現只有 Bearer-token 路徑
  （`LocalApiAuth.java`/`LocalApiAuthHandler.java`）才會丟的 `Asset.Token.Invalid`
  錯誤碼。目前判斷是裝好的舊版 exe 邏輯跟現在看到的原始碼不一致（見上面 Bug 2
  的版本沒對齊），但沒有逐行比對舊 exe 反組譯確認，只是重編譯安裝後問題確實消失。
  下次如果同樣的錯誤訊息又出現在**剛重新編譯安裝的新版**上，才需要真的回頭深挖
  `AutoReportService`/`AssetApi`/`AipowerClient` 密碼模式路徑為什麼會冒出這個
  Bearer-token 專屬錯誤碼。

  **解密 `agent_credentials.bin` 看實際存的憑證/模式**（DPAPI 綁定目前 Windows
  帳號，PowerShell 用 `System.Security.Cryptography.ProtectedData` 就能解，不用
  跑 Flutter 程式）：
  ```powershell
  Add-Type -AssemblyName System.Security
  $path = "$env:APPDATA\flutter_tray_app\agent_credentials.bin"
  $bytes = [System.IO.File]::ReadAllBytes($path)
  $plain = [System.Security.Cryptography.ProtectedData]::Unprotect(
      $bytes, $null, [System.Security.Cryptography.DataProtectionScope]::CurrentUser)
  [System.Text.Encoding]::UTF8.GetString($plain) | ConvertFrom-Json | ConvertTo-Json
  ```

- **2026-07-23（再續7）** ✅ **資產管理頁面命令框加「AI 提示詞」模式（呼叫既有 AiProxy），
  外加一個很重要的踩坑記錄：`TcAiProxySetting` 改值不是即時生效，也不是「重啟就生效」，
  而是「重啟後 5 秒、只同步一次」。**

  **功能本身**：`buildDetailRow` 展開區統一成兩欄（離線機器也一樣，只是「Shell 命令」
  按鈕 disable），新增模式切換鈕（Shell／AI），AI 模式呼叫既有的
  `POST /aipower/v1/chat/completions`（這個 Docker 部署自己就有活著的 AiProxy，
  session cookie 認證即可，**前端直接 `fetch`，完全沒有加任何後端 Java**）。回覆顯示
  在同一塊 `.terminalPane` 裡（跟 shell 命令結果共用視覺樣式）。

  **AiProxy 設定同步機制（反組譯 `AiProxyHome`/`AiProxyStartupListener` 才挖出來的
  真相，不要憑感覺猜）**：
  - `AiProxyHome.getConfig(key)` 讀的是**行程內記憶體** `ConcurrentHashMap`
    （`configCache`），**不是每次查 DB**——跟 `ecp-deepseek` skill 記載的另一個部署
    （`DeepSeekHome.getConfig` 每次 `SELECT`、真的即時生效）行為完全不同，**兩個是不同
    的實作，不能把那邊的經驗直接套到這裡**。
  - `AiProxyStartupListener.contextInitialized()`：Tomcat 啟動時先讀
    `/app/apache-tomcat/extension/aipower/config/aiproxy.properties` 檔案灌進快取，
    然後**排程一個 5 秒後才執行、且只執行一次的非同步工作**（`scheduler.schedule(...,
    5, SECONDS)`，不是 `scheduleAtFixedRate`）去查 `TcAiProxySetting`、把 DB 的值蓋過
    去（`AiProxyHome.setConfigDirect`）。
  - **結論**：改 `TcAiProxySetting` 的值之後，一定要 `docker restart aipower-app`
    才會生效，而且**要等重啟後至少 5-10 秒**再驗證，太快測會看到舊值（誤判成「沒生效」）；
    **改完之後如果同一個 Tomcat 生命週期內又手動改一次 DB，不會再被撿到**，因為那個
    排程只跑一次——這次就是這樣自己搞亂過一次（先改 `deepseek-v4-pro`、重啟後又手滑
    改了兩次垃圾值想測試「是不是即時生效」，結果垃圾值永遠沒生效，因為 5 秒同步窗口
    早就過了，行程記憶體裡其實一直是重啟當下抓到的 `deepseek-v4-pro`，DB 反而變成
    跟記憶體對不上的髒值，最後要手動把 DB 改回來對齊）。
  - **正確的驗證方法**：不要用聊天內容去猜是哪個模型回的（`/v1/chat/completions` 回應的
    `model` 欄位固定回 `"ecp-ai"`，是 AiProxy 自己的包裝身份，看不出底層真正呼叫的
    DeepSeek 模型）。改用 `GET /aipower/v1/health`（不用登入也能打），回應
    `{"model":"...","status":"ok"}` 裡的 `model` 欄位就是直接讀 `AiProxyHome.getConfig
    ("aiproxy.deepseek.model")` 的當下記憶體值，一次就能確認到底生效了沒。
  - **部署動作**：`docker exec aipower-mariadb mariadb -uroot -p<DB_PASSWORD> default -e
    "UPDATE TcAiProxySetting SET FValue='deepseek-v4-pro' WHERE FKey='aiproxy.deepseek.model';"`，
    這組設定也被 LINE 客服台的 AI 建議回覆功能共用（`cbm-agent-ai-suggest` 那條線，
    走同一個 `/aipower/v1/chat/completions`），改動是全局的，刻意的。
  - **反組譯技巧補充**：這幾個 `com.chainsea.ecp.aiproxy.*` class 不在任何 vendor jar
    裡，是直接放在 `WEB-INF/classes` 的自訂編譯類別（`docker cp` 單一 class 檔出來，
    用 `javap -p -c -constants` 看常數池字串＋bytecode 呼叫順序，不需要完整反編譯器
    也能看懂「這個方法到底讀了哪個 config key、有沒有快取」這類問題）。

## Pi 聊天框：呼叫同機 aipower-pi-agent 容器（2026-07-26，全新環境上的第一個功能）

背景：這台 `C:\aipower` compose 專案除了 `db`/`app` 之外，使用者自己另外加了第三個 service
`pi-agent`（`build: ./pi-agent`，container `aipower-pi-agent`），裝的是
**`@earendil-works/pi-coding-agent`**（一支跟 Claude Code/opencode 類似的 AI coding agent
CLI，`node:24-bookworm-slim` 底圖），環境變數帶 `DEEPSEEK_API_KEY`，`volumes` 把整個
`C:\aipower`（compose 專案根目錄本身）掛進容器的 `/workspace`，且掛了
`C:\Users\HCH\.claude\skills:/root/.claude-skills:ro`（唯讀，讓這支 agent 也能讀到跟這裡
一樣的 skills）。原本 `CMD ["tail","-f","/dev/null"]` 只是讓容器活著、沒有任何服務在監聽，
需求是「在 aipower 網頁裡做一個聊天框，能呼叫這支 pi-agent」。

**使用者已確認的關鍵設計決策：聊天框呼叫 pi 時開放完整工具能力（bash/read/edit/write），
不加 `--no-tools`**——代表任何能打開這個聊天框頁面的登入帳號，等於能請 pi 讀寫/執行
`/workspace`（=host 上的 `C:\aipower`）底下的任何東西，這是刻意的取捨，不是疏漏。

### 架構：aipower-app 沒有 docker.sock，改幫 pi-agent 加一個內部 HTTP wrapper

`aipower-app`（Java/Tomcat 容器）沒有掛 docker socket，無法 `docker exec` 進
`aipower-pi-agent`。兩個容器都在同一個 `aipower-net` bridge network、彼此可用容器名稱
（`pi-agent`/`aipower-pi-agent`）互相解析，所以改成幫 pi-agent 容器裝一支**常駐**的
Node.js HTTP wrapper（`C:\aipower\pi-agent\server.js`，只用內建 `http`/`child_process`，
零額外依賴），取代原本的 `tail -f /dev/null`：

- `Dockerfile` 的 `CMD` 改成 `["node", "/workspace/pi-agent/server.js"]`——因為
  `docker-compose.yml` 本來就有 `.:/workspace` 這個掛載，`server.js` 放在
  `C:\aipower\pi-agent\server.js` 這個 host 路徑，容器裡透過這個既有掛載點就看得到，
  **改 `server.js` 內容完全不用重建 image**，只有改 `Dockerfile` 本身（例如這次改 CMD）
  才需要 `docker compose build pi-agent && docker compose up -d pi-agent`。
- `POST /chat {message, sessionId}` → `spawn('pi', ['-p', message, '--provider','deepseek',
  '--model','deepseek-v4-flash','--mode','text','--no-approve','--session-id', sessionId],
  {cwd:'/workspace', stdio:['ignore','pipe','pipe']})`，抓 stdout trim 後回傳
  `{reply: "..."}`。`GET /health` 回 `{status,provider,model}`。

### ⚠ 致命坑：Node 預設的 stdin pipe 讓 `pi -p` 無限期卡死，`docker exec` 直接測完全看不出來

第一版 `spawn('pi', args, {cwd:'/workspace'})`（沒指定 `stdio`）——用 `docker exec
aipower-pi-agent pi -p "..."` 直接測完全正常（幾秒回應），但透過 HTTP wrapper 呼叫
（Node 當 PID 1 自己 spawn）卻無限期卡住，240 秒逾時後 kill 都還在 `epoll_wait` 睡著。

**根因**：Node 的 `child_process.spawn()` 預設會給子行程一個「開著、但永遠不會收到 EOF」
的 stdin pipe（因為父行程沒有寫入也沒有關閉它）。`pi` 在 `-p` 非互動模式下仍然會嘗試讀
stdin（疑似用來偵測有沒有人用 `cat file | pi` 這種方式額外餵資料進來），沒等到 EOF
就會卡住等。**`docker exec CONTAINER cmd`（沒加 `-i`）給的 stdin 是直接 closed/EOF**，
這正是為什麼 `docker exec` 直接測試完全測不出這個坑——兩種呼叫方式的 stdin 語意不同。

**修法**：`spawn('pi', args, {cwd:'/workspace', stdio:['ignore','pipe','pipe']})`——
明確把 `stdio[0]` 設成 `'ignore'`，等同給子行程一個立即 EOF 的 `/dev/null`，行為對齊
`docker exec` 沒加 `-i` 的情況。**教訓：任何要從 Node/Java 等語言用 `spawn`/`ProcessBuilder`
包一支 CLI 工具、且該 CLI 支援「讀取額外 stdin 輸入」這種功能時，一定要明確設定 stdin
為關閉/EOF，不要依賴語言預設值——用 `docker exec` 手動測試這條路徑本身無法暴露這個坑，
必須實際透過同一種呼叫方式（spawn）驗證。**

### Java 端：`PiAgentConsoleServlet` + 獨立的 `PiAgentStartupListener`

跟 `LineChatConsoleServlet`/`AssetConsoleServlet` 一樣是原生 `HttpServlet`（不透過
Quicksilver 六大核心），掛在 `/aipower/pi/console`，原始碼在這個 skill 目錄下的
`pi-agent-integration/src/com/chainsea/ecp/piagent/`。**但因為這台容器是全新安裝（見上面
的路徑警告），沒有借用（其實不存在的）`LineIntegrationStartupListener`**，而是新建一個
獨立的 `PiAgentStartupListener implements ServletContextListener`，透過 `web.xml` 的
`<listener>` 明確掛載（`web.xml` 原本只有 `<distributable/>`，改完的版本存在
`pi-agent-integration/web.xml`）——這代表**沒有任何既有 servlet/listener 可以借用時，
正確的做法是自己建一個 `<listener>` 掛進 `web.xml`，不是硬要塞進某個不存在的舊檔案**。
登入驗證邏輯（`getCurrentOnlineUser`，抄 `OnlineUserHome.getService().getItem(session)`
這個官方 API）直接內嵌在 servlet 裡，刻意不依賴 `LineIntegrationHome`（那個 class 目前
也不存在於這個容器）。

對話記憶：`Map<accountId, ChatState>` 純記憶體（Tomcat 重啟清空，比照
`agentReadyStatus`/`ONLINE_AGENTS` 這幾個既有先例），但 `pi --session-id` 自己的 session
檔案存在 `pi_agent_home` 這個獨立 docker volume，不受 aipower-app 重啟影響——「畫面上的
對話紀錄」跟「pi 實際記得的上下文」是兩回事。

### ⚠ 又踩到一次：raw HttpServlet 前端 JS 用相對路徑 `fetch('send')`，在瀏覽器裡才會炸

`curl http://host/aipower/pi/console/send`（絕對路徑）完全正常，但瀏覽器裡點擊送出卻回
「網路錯誤：`<!DOCTYPE` is not valid JSON」——因為 `fetch('send')`（相對路徑，無前導斜線）
是相對**目前網址去掉最後一段路徑**解析，目前網址是 `.../aipower/pi/console`（無結尾斜線），
所以 `send` 解析成 `.../aipower/pi/send`，不是 `.../aipower/pi/console/send`！**只有透過
真實瀏覽器點擊才會暴露這個 bug，curl 用絕對路徑測試完全看不出來。** 修法：JS 裡建一個
`const BASE='/aipower/pi/console'`，所有 `fetch` 呼叫改成 `fetch(BASE+'/send')` 這種絕對
路徑寫法，不要依賴相對路徑解析。**這個坑的教訓比這次事故本身更重要：`AssetConsoleServlet`/
`AcdAdminServlet`/`ChatReportServlet` 這些姊妹 servlet 的 `PAGE_HTML` 全部都用一樣的相對
路徑 `fetch('assets')`/`fetch('agents')` 風格寫法，理論上都有同一個潛在 bug，只是它們的
瀏覽器實測大多因為 CDP 連不上而沒有真正走完（見這些檔案開頭註解反覆出現「這次沒能在瀏覽器
裡實際驗證」）——之後任何時候真的把這些頁面走通瀏覽器測試，如果也遇到「API 回應變成 HTML
500 頁」，第一個該懷疑的就是這個相對路徑解析問題，不是後端邏輯。**

### `scripts/deploy-servlet.sh` 本身的另一個真實 bug：只複製跟檔名同名的 class，漏掉內部類別

`PiAgentConsoleServlet` 裡有一個 `private static class ChatState`，javac 編譯出
`PiAgentConsoleServlet.class` 之外還會多產生一個 `PiAgentConsoleServlet$ChatState.class`。
舊版 `scripts/deploy-servlet.sh` 的部署迴圈只用 `basename "$f" .java` 算出唯一一個
`$base.class` 路徑去 `docker cp`，完全沒考慮內部類別/匿名類別/lambda 合成類別——結果
`docker cp` 只複製了 `PiAgentConsoleServlet.class`，執行時一叫到用 `ChatState` 的方法就
`NoClassDefFoundError`。**這個 bug 對過去所有姊妹整合都潛在存在，只是那些檔案可能剛好沒
真正的具名內部類別（很多是用 `Map`/`ConcurrentHashMap` 存簡單型別，沒包裝成自己的
class）才沒踩到。** 已經在腳本本體修好（改成 glob `$base.class` 與 `$base\$*.class`
全部複製），這是永久修正，之後任何有內部類別的檔案都不會再漏。

### 部署與驗證指令

```bash
# 1. pi-agent 容器：改 Dockerfile CMD 才需要 build（改 server.js 內容不用）
cd C:/aipower && docker compose build pi-agent && docker compose up -d pi-agent

# 2. 用一次性 alpine 容器測內部網路連通性（不用進 aipower-app，它沒裝 curl）
docker run --rm --network aipower_aipower-net alpine sh -c \
  "apk add --no-cache curl -q; curl -s http://pi-agent:4173/health"

# 3. Java 端沿用既有 scripts/deploy-servlet.sh（這次額外确认了它也能正確處理「全新容器、
#    WEB-INF/classes 目錄本身都不存在」的第一次部署情境，已修好 mkdir -p 那段）
bash scripts/deploy-servlet.sh pi-agent-integration/src/com/chainsea/ecp/piagent/PiAgentConsoleServlet.java \
  pi-agent-integration/src/com/chainsea/ecp/piagent/PiAgentStartupListener.java

# web.xml 沒有自動化流程，第一次要手動 docker cp 修改過的版本進去（見
# pi-agent-integration/web.xml，之後如果 servlet 數量增加、想改回借用單一
# StartupListener 模式，可以參考 line-integration/ 那套寫法）
docker cp pi-agent-integration/web.xml aipower-app:/app/apache-tomcat/webapps/aipower/WEB-INF/web.xml
docker restart aipower-app
```

**TsMenu**：這台全新資料庫目前只有「辦公自動化」（`002`）與「系統管理」（`003`）兩個
頂層分類，沒有舊環境記載的「開發→工具」（那是舊環境自己建的，這裡不存在）。「Pi 聊天室」
直接掛在「辦公自動化」底下現有的 `002.001` 子目錄（`FTreeSerial='002.001.002'`，
`FType='ExternalPage'`，`FExternalPageUrl='/aipower/pi/console'`，同源相對路徑——沿用
「iframe 內嵌頁面網址一律用相對路徑，不要寫死網域」這條全部姊妹整合都驗證過的教訓）。

**端到端驗證**：Playwright 實測通過——登入（見下面密碼重設）→ 開新分頁直接打
`/aipower/pi/console`（不要在 MainFrame 內用 `browser_navigate` 打這種原生 servlet 網址，
會弄壞 MainFrame 內部路由狀態，見已知踩坑章節；用 `browser_tabs action:new` 開獨立分頁，
或透過點擊側邊選單讓框架自己在 MainFrame 內開新 iframe）→ 送出訊息「你好,請用一句話簡短
自我介紹」拿到 pi 的真實回覆（提到能讀寫檔案/執行命令/操作 Docker/瀏覽器，證實真的讀到
`/workspace` 環境）→ 再問「我剛剛問你的第一個問題是什麼?」正確答對，證實 `--session-id`
跨兩次獨立 HTTP 請求的上下文記憶有效。也確認選單項目能正確在 MainFrame 內以 iframe 開啟。

### 2026-07-26（續）：使用者要求聊天框能查 aipower-app 的錯誤 → 掛 `docker.sock` 給完整 Docker 控制權

使用者的實際需求不只是「聊天」，是「能查 docker 內 aipower 的錯誤」。原本 pi-agent 容器
`/workspace` 只掛了 compose 專案目錄本身，沒有 docker CLI、也沒有 docker.sock，看不到
`aipower-app` 的 log。問使用者「只掛 log 目錄唯讀」vs「掛 docker.sock 給完整控制權」兩個
風險程度不同的選項，**使用者選了完整控制權**。做法：

1. `pi-agent/Dockerfile` 的 apt 套件清單加 `docker.io`（Debian bookworm 官方套件，內含
   docker CLI，不需要另外加 Docker 官方 apt repo；不會啟動 daemon，容器只是拿它的
   client 二進位檔）。
2. `docker-compose.yml` 的 `pi-agent` service `volumes` 加一行
   `/var/run/docker.sock:/var/run/docker.sock`——Docker Desktop（Windows，WSL2 backend）
   下這個標準 Linux 掛法直接可用，不用轉換成 `//./pipe/docker_engine` 那種 Windows npipe
   語法（那是給「host 本身」的 docker CLI 連 daemon 用的，容器內部走的是 WSL2 VM 裡的
   標準 unix socket）。
3. 兩者都動了 `Dockerfile`，一定要 `docker compose build pi-agent && docker compose up -d
   pi-agent`（跟只改 `server.js` 內容不同，不能只用 `docker restart`）。
4. 驗證：`docker exec aipower-pi-agent docker ps`／`docker logs aipower-app` 直接能查到
   host 上所有容器，確認掛載生效。

**風險備忘**：這代表 pi-agent 容器現在等同 host Docker 的 root 權限（能看/管理/刪除
*任何*容器，不只 aipower-app），而觸發它的聊天框只驗證「是不是登入過的 aipower 帳號」，
沒有更細的角色權限——跟這個環境其他工具（`AgentControlEndpoint` 執行任意 shell 指令、
`AcdAdminServlet` 任何登入帳號都能操作）風險胃口一致，是使用者自己權衡後的選擇，不是
預設值，之後若要收斂權限（例如只給讀 log、拿掉 docker.sock），移除
`docker-compose.yml` 那行掛載、Dockerfile 拿掉 `docker.io` 即可，`pi` CLI 本身的行為
完全不受影響（docker 只是它可呼叫的其中一個 bash 指令）。

**端到端驗證**：透過聊天框問「用 docker logs 查一下 aipower-app 容器最近有沒有出現什麼
錯誤或例外」，pi 自己決定呼叫 `docker logs aipower-app`（無需任何額外提示詞引導），
正確整理出當時真實存在的三個問題（`ServiceRequestUpgrade` NPE、登入 `SystemException`、
Tomcat ThreadLocal 洩漏警告），包含檔案位置與根因分析，證實 docker.sock 授權跟聊天框
本身的整合完全生效。

## 讓 pi-agent「控制整個系統」：Docker + DB + aipower API 三層 + LLM Wiki 當 RAG（2026-07-28）

使用者要求「讓 pi agent 能控制整個系統」——問清楚範圍後，使用者選了全部三層都要：
基礎設施層（Docker，本來就有 `docker.sock`，見上一節）、直接改資料庫、
應用程式層（走 aipower 標準 Action/API，非只操作 DB）。額外又要求接上本機的
LLM Wiki 桌面應用當 pi 的 RAG 知識庫。

### 1. 應用程式層：建一個專屬服務帳號 `pi-agent`，角色「系統管理」

不重用 `administrator` 真人帳號，理由是可獨立撤銷/audit（跟 Flutter 用的
`agent`/`asset-agent` 服務帳號是同一慣例）。**這台全新環境沒有 memory 舊筆記裡
那套「員工身份」recipe 的既有範例可抄，改成直接反查 `administrator` 自己在
`TsAccount`/`TsUser`/`TsAccountIdentity`/`TsRoleUser` 的真實關聯**（比自己重新
推導 identity chain 可靠）：

```sql
-- administrator 的實際關聯（本環境反查得到，跟 2026-07-22 那份舊 recipe 用的
-- 「員工」身份類型不同，這裡走「職務」）：
-- TsUser.FAccountId 是 NULL（不是連回 TsAccount，帳號<->員工是靠
-- TsAccountIdentity.FEntityId 指過去，不是靠 TsUser.FAccountId）
-- TsAccountIdentity.FIdentityTypeId = 564cf69e-...（職務），FEntityId = TsUser.FId
-- TsRoleUser.FUserId = TsUser.FId（不是 TsAccount.FId），FRoleId = 系統管理角色

INSERT INTO TsAccount (FId, FName, FLoginName, FPassword, FCreateTime, FEnabled)
VALUES ('<新UUID>', 'Pi Agent 服務帳號', 'pi-agent', '<PasswordUtil hash>', NOW(3), b'1');

INSERT INTO TsUser (FId, FName, FAccountId, FEnabled, FLoginName, FDepartmentId, FIndex, FIsDuty)
VALUES ('<新UUID>', 'Pi Agent 服務帳號', NULL, b'1', 'pi-agent', '<任一真實 TsDepartment.FId>', 2, b'0');

INSERT INTO TsAccountIdentity (FId, FName, FAccountId, FIdentityTypeId, FEntityId, FDefault, FIsMainDuty, FIndex)
VALUES ('<新UUID>', 'Pi Agent 服務帳號', '<TsAccount.FId>', '564cf69e-76d6-4baf-b584-6e04c2911dae', '<TsUser.FId>', b'1', b'1', 1);

INSERT INTO TsRoleUser (FRoleId, FUserId)
VALUES ('00000000-0000-0000-1004-000000000002' /*系統管理*/, '<TsUser.FId>');
```

密碼雜湊沿用 `scripts/reset_admin_password.ps1` 的 `PasswordUtil.encode()` 演算法
（見上面「密碼重設」章節）。**改完一定要 `docker restart aipower-app`**（帳號/
身份查詢是 JVM 記憶體快取，這是本環境第三次印證這個坑）。

**登入 API 有一個必填、容易漏掉的參數**：`Qs.OnlineUser.login.data` 沒帶
`language` 會回 `"Language 'null' is invalid."`，帶 `"language":"zh-tw"` 就過——
即使 `TsAccount.FLanguage` 欄位本身是 NULL 也一樣，這個參數是 request body 的，
跟帳號自己存的語言設定是兩回事。登入成功回應本身是 `{}`（不是回傳 token/身份
資訊），驗證是否真的登入成功要看有沒有拿到 `Set-Cookie: JSESSIONID=...`，並且
再用這個 cookie 呼叫任一個真實的 `getListData.data` 端點確認有資料權限。

驗證方式：`curl -c cookie.txt` 登入 → `curl -b cookie.txt` 呼叫
`Qs.Account.getListData.data`，能看到帳號清單代表角色權限（系統管理）真的生效，
不是只登入成功但沒權限。

### 2. 直接操作資料庫：給 pi-agent 容器裝 `mariadb-client` + 連線環境變數

`pi-agent/Dockerfile` apt 套件加 `mariadb-client`；`docker-compose.yml` 的
`pi-agent` service `environment` 加 `AIPOWER_DB_HOST=db`／`AIPOWER_DB_PORT=3306`／
`AIPOWER_DB_USER=root`／`AIPOWER_DB_PASSWORD=<DB_PASSWORD>`／`AIPOWER_DB_NAME=default`
（`db` 是 compose service 名稱，pi-agent 容器跟 db 在同一個 `aipower-net` bridge
network，直接用服務名稱解析，不用查 IP）。`docker compose build pi-agent &&
docker compose up -d pi-agent` 套用。

同時也把 `AIPOWER_BASE_URL=http://app:22821/aipower`／`AIPOWER_APP_LOGIN`／
`AIPOWER_APP_PASSWORD` 一起塞進環境變數，讓 pi 不用問使用者要連線資訊就能自己
組出登入+呼叫 API 的指令。

### 3. 把這三層寫成一份文件放進 `/workspace`，讓 pi 自己讀得到

寫在 `D:\aipower\AIPOWER-CONTROL.md`（compose 專案根目錄，本來就整個掛進
pi-agent 容器的 `/workspace`），內容涵蓋三層的連線資訊、`curl` 呼叫範例、以及
「改 DB 裡的 metadata 要重啟 `aipower-app` 才生效」這個本環境反覆出現的陷阱。
**不用額外寫程式碼餵給 pi，直接放一份 markdown 文件在它看得到的工作目錄，
它會自己讀、自己照著做**——這次端到端測試（透過真實的 `/aipower/pi/console/send`
呼叫，不是只在容器內單獨測 `pi -p`）證實 pi 真的會自己選對指令（docker
ps/logs、mariadb 查詢、curl 呼叫 API 三者都選對），不需要在提示詞裡手把手教它
每個指令怎麼下。

⚠️ **風險提醒（已跟使用者說明過，是他選的）**：`pi-agent` 帳號是系統管理員角色 +
DB root 密碼 + `docker.sock`，三者疊加代表 pi 對整套 aipower 部署（資料、設定、
容器本身）幾乎有完全控制權，且觸發它的聊天框（`/aipower/pi/console`）只驗證
「是不是登入過的 aipower 帳號」，沒有更細的角色限制——跟這個環境其他工具
（`AgentControlEndpoint`、`AcdAdminServlet`）的風險胃口一致。

### 4. 接上 LLM Wiki 桌面應用當 pi 的 RAG（跨 Windows host ↔ Linux 容器，`connect-llm-wiki` skill 的坑都要改一版）

`connect-llm-wiki` skill 原本假設 pi 直接跑在 Windows host 上，能直接用
`%LOCALAPPDATA%\LLM Wiki\mcp-server` 路徑、`127.0.0.1:19828`。但這裡的 pi 是跑在
Linux 容器（`aipower-pi-agent`）裡，需要三個額外轉換：

1. **`127.0.0.1:19828` 對容器來說是容器自己，不是 host**——LLM Wiki 桌面 app
   跑在 Windows host 上，容器要用 Docker Desktop（Windows/WSL2 backend）自動
   提供的 `host.docker.internal` 才能連到 host 的 loopback，不需要額外
   `extra_hosts` 設定（`getent hosts host.docker.internal` 直接解析得到）。
2. **LLM Wiki 內建的 MCP server（`%LOCALAPPDATA%\LLM Wiki\mcp-server`）是
   Windows host 上的檔案，容器看不到**——用 `robocopy` 把整個資料夾（含
   `node_modules`，純 Node 腳本、不需要重新編譯）複製進 compose 專案目錄底下
   （例如 `D:\aipower\pi-agent\llm-wiki-mcp-server`），因為那個目錄本來就整個
   掛進容器的 `/workspace`，複製過去容器立刻就看得到，不用 rebuild image。
3. **`pi-mcp-adapter` 擴充套件要在容器裡裝**：`docker exec aipower-pi-agent pi
   install npm:pi-mcp-adapter`——它其實裝到 `/root/.pi/agent/npm/node_modules/`，
   剛好在既有的 `pi_agent_home` 持久化 volume 裡，容器重建也不會消失。

`~/.config/mcp/mcp.json`（`pi-mcp-adapter` 讀取的設定檔路徑）指向剛複製進去的
腳本＋`host.docker.internal` 網址：
```json
{
  "mcpServers": {
    "llm-wiki": {
      "command": "node",
      "args": ["/workspace/pi-agent/llm-wiki-mcp-server/dist/src/index.js"],
      "env": {
        "LLM_WIKI_API_TOKEN": "<Settings -> API + MCP 裡的 token>",
        "LLM_WIKI_API_BASE_URL": "http://host.docker.internal:19828"
      }
    }
  }
}
```

⚠ **這個坑最重要**：`/root/.config/mcp/` **不在任何掛載的 volume 上**，只有
`/root/.pi/agent`（`pi_agent_home`）、`/workspace`、`/root/.claude-skills`、
`/var/run/docker.sock` 這四個路徑是持久化的——單純用 `docker exec` 寫進
`mcp.json` 測試沒問題，但下次 `docker compose build`/`up -d` 重新建立容器就會
整個消失。**正確做法是把 mcp.json 的內容用環境變數＋一支 entrypoint 腳本，在
每次容器啟動時重新產生**：

```dockerfile
# Dockerfile：CMD 從直接跑 server.js 改成先跑 entrypoint 腳本
CMD ["sh", "/workspace/pi-agent/docker-entrypoint.sh"]
```

`docker-entrypoint.sh`（放在 `D:\aipower\pi-agent\`，因為整個資料夾都在
`/workspace` 掛載範圍內，改內容不用 rebuild，只有改 `Dockerfile` 本身的 CMD
才需要）：

```sh
#!/bin/sh
set -e
mkdir -p /root/.config/mcp
if [ -n "$LLM_WIKI_API_TOKEN" ] && [ -n "$LLM_WIKI_API_BASE_URL" ]; then
  cat > /root/.config/mcp/mcp.json <<EOF
{
  "mcpServers": {
    "llm-wiki": {
      "command": "node",
      "args": ["/workspace/pi-agent/llm-wiki-mcp-server/dist/src/index.js"],
      "env": {
        "LLM_WIKI_API_TOKEN": "$LLM_WIKI_API_TOKEN",
        "LLM_WIKI_API_BASE_URL": "$LLM_WIKI_API_BASE_URL"
      }
    }
  }
}
EOF
fi
exec node /workspace/pi-agent/server.js
```

`docker-compose.yml` 的 `pi-agent` service 加 `LLM_WIKI_API_TOKEN`／
`LLM_WIKI_API_BASE_URL=http://host.docker.internal:19828` 環境變數。**這個
模式（易變/敏感設定用環境變數驅動、entrypoint 每次啟動重新產生檔案，而不是
`docker exec` 手動改容器內某個非掛載路徑的檔案）值得記住，任何「pi 或其他
容器需要一份寫在非 volume 路徑上的 dotfile/設定檔」的需求都應該照這個套路，
不要只做一次性的 `docker exec` 寫入就以為完工。**

**驗證方式（且要驗證「重建容器後還在」，不是只驗證「剛裝好那次」）**：
`docker compose build pi-agent && docker compose up -d pi-agent`（完整重建，
不是 `docker restart`）之後，確認 `docker logs aipower-pi-agent` 有印出
entrypoint 產生設定檔的訊息、`pi-mcp-adapter` 仍在（volume 保留）、再跑一次
`pi -p --no-session "Call the llm_wiki_status MCP tool..."` 確認 MCP 連線
依然正常——這三者都要重新確認過一次才算真的做到「durable」，只測「重建前」
不能代表「重建後」也一樣。

### 5. 讓 pi 自己知道「LLM Wiki 就是我的 RAG」：寫進全域 `AGENTS.md`

pi-coding-agent 會在啟動時載入 `AGENTS.md`/`CLAUDE.md`（`pi --help`/README 有
記載：`~/.pi/agent/AGENTS.md`（全域）+ 從 cwd 往上找的專案層級檔案，全部檔案
會被串接起來）。**`~/.pi/agent/AGENTS.md` 剛好也在 `pi_agent_home` 持久化
volume 裡**，寫在這裡比寫在 `/workspace/AGENTS.md`（專案層級）更適合這次需求
（「pi 這個 agent 本身該怎麼使用它的知識來源」是全域行為，不是這個特定
compose 專案的規則）：

```
# 知識來源：LLM Wiki 就是你的 RAG
（列出 llm_wiki_llm_wiki_status/projects/set_project/search/files/read_file/
chat/graph/reviews/rescan_sources 這幾個實際可用的工具名稱——MCP adapter 會把
server key "llm-wiki" 疊加在工具名前面，實際名稱是雙重前綴
`llm_wiki_llm_wiki_xxx`，不是只有一層 `llm_wiki_xxx`，寫文件時要用實測到的
真實名稱，不要憑印象猜）

## 使用原則
1. 回答不確定/屬於使用者專案領域知識的問題前，先查 llm_wiki_llm_wiki_search
   或 llm_wiki_llm_wiki_chat，優先度高於自己訓練資料裡的印象。
2. 這個 MCP 連線是唯讀的，沒有寫入/新增工具，查無資料要老實說，不要編造；
   使用者要求「記錄下來」也要老實說自己沒有寫入能力。
```

驗證方式：不帶任何提示詞明講「用 llm wiki」，直接問 pi「自我介紹一下你有什麼
知識來源」，正常應該會主動列出 LLM Wiki 是它的「最重要的專案知識來源」且
優先於訓練資料——這樣才算真的內化成行為準則,而不是只在被明確要求時才想到用。

## 2026-07-29：`10.145.119.19` 上的 vLLM 換成 v2 模型，順便修好「gigabyte」provider 一直沒真的能用的隱藏 bug

使用者反映「docker 中的 aipower 呼叫 10.145.119.19 的模型換了，他現在是 v2 的」。查證
`curl http://10.145.119.19:8182/v1/models` 確認 served-model-name 從 `qwythos-9b-awq`
變成 **`qwythos-9b-v2-awq`**（`root` 也變成 `/models/Qwythos-9B-v2-AWQ`，見
[[vllm-qwythos-deploy]] skill 那台機器）。這個模型只有一個地方在用：「Pi 聊天室」
（`/aipower/pi/console`）下拉選單裡的「本地 vLLM qwythos-9b」選項（`providerPreset:
"gigabyte"`）。

**兩個檔案要改**（都在 `D:\aipower\pi-agent\`，bind-mount 進 `aipower-pi-agent` 容器
的 `/workspace/pi-agent/`，改完不用 rebuild，只要 `docker restart aipower-pi-agent`）：
1. `extensions/vllm-gigabyte-provider.ts`：`models[0].id` 從 `qwythos-9b-awq` 改成
   `qwythos-9b-v2-awq`（連帶 `name` 欄位跟頂層 provider `name` 註記一併改，純顯示用）。
2. `server.js`：`PROVIDER_PRESETS.gigabyte.model` 同步改成 `qwythos-9b-v2-awq`。

**⚠️ 順便發現一個從一開始就存在、跟這次換模型無關的獨立 bug**：`server.js` 的
`runPi()` 組 `pi` 指令的 `args` 陣列裡**從來沒有帶 `--extension` 參數**，代表
`gigabyte-vllm` 這個自訂 provider（定義在 `vllm-gigabyte-provider.ts`，靠 pi 的
`registerProvider()` API 註冊）**從來沒有被真正載入過**——`pi` 的 extension
自動探索只掃 `<cwd>/.pi/extensions` 跟 `~/.pi/agent/extensions`（反編譯
`core/extensions/loader.js` 的 `discoverExtensions()`/`localExtDir`/`globalExtDir`
確認），兩者都不是 `vllm-gigabyte-provider.ts` 實際的位置
（`/workspace/pi-agent/extensions/`），所以直接 `docker exec aipower-pi-agent pi
--provider gigabyte-vllm ...`（沒加 `--extension`）一律回
`Error: Unknown provider "gigabyte-vllm"`。**這代表「本地 vLLM」這個選項在 Pi
聊天室裡自建立以來可能從沒有真的成功過一次**，這次順手在 `runPi()` 加了
`extension` 參數（`PROVIDER_PRESETS.gigabyte.extension` 帶絕對路徑，`args` 陣列
只有這個 preset 有值時才 push `--extension <path>`），deepseek 那組不受影響。

**驗證方式**：先用 `docker exec aipower-pi-agent sh -c "cd /workspace && pi -p '...'
--provider gigabyte-vllm --model qwythos-9b-v2-awq --extension
/workspace/pi-agent/extensions/vllm-gigabyte-provider.ts --no-session"` 手動測通
（拿到 `我是 Qwythos...` 的正確回覆），改完 `server.js` 後 `docker restart
aipower-pi-agent`，直接對 `POST http://localhost:4173/chat`（容器內部埠，跳過
Java 那層）送 `{"providerPreset":"gigabyte", ...}` 驗證整條「網頁→
`PiAgentConsoleServlet`→`server.js`→`pi`→vLLM v2」路徑，回應 200 且真的拿到
（雖然措辭有點語無倫次的）回覆，證實 pipeline 已通。**教訓**：`vllm-qwythos-deploy`
skill 提醒過「換模型不會自動同步 opencode.jsonc」，這裡是同一類問題的另一個
受害者——任何寫死 served-model-name 字串的下游整合（不只 opencode），換模型後都要
逐一巡一遍，不能只改一個地方就假設全部同步。

## 相關 Skill
- `ecp-unit-setup-via-ui`：Unit/Page/Field/Menu 完整元資料規則、如何從零建一個標準單元
- `ecp-pwd`：本機 embedded MariaDB（非 Docker）版本的密碼重設
- `ecp-schema`：查 Chainsea/aipower 資料庫結構的通用方法
- `connect-llm-wiki`：pi/opencode 跑在 host 上（非容器）時接 LLM Wiki MCP 的原始版本
  流程，這裡的第 4 節是它的 Docker 容器化改版

---

## Conformance Addendum

## When to Use
Manage the local D:\aipower Docker deployment (aipower-app + aipower-mariadb + aipower-pi-agent containers on this Windows machine's Docker Desktop; compose folder path has moved, verify with Test-Path first) — restart/ relogin, reset admin password, clean up orphaned Unit/Page/Field/Form/List/ Privilege metadata, menu-hiding field gotchas, building a standard six-core Unit whose Service layer calls an external HTTPS API, and compiling/ deploying the hand-written LineChatConsoleServlet/LineWebhookServlet (/aipower/line/console, a raw HttpServlet outside the Unit/Page/Menu chain). Also covers the aipower-pi-agent container (pi-coding-agent + DeepSeek, exposed as "Pi 聊天室" at /aipower/pi/console) with full system control and MCP-based RAG wiring to a host LLM Wiki app. Use when `docker ps` shows aipower-app/aipower-mariadb/aipower-pi-agent. NOT the 10.145.119.19 remote instance or the native C:\com\chainsea / C:\Lab2 installs.

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
