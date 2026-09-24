---
name: "aipower-module-cleanup-and-debug"
description: "在 AiPower/Quicksilver 框架系統中安全刪除選單模組與對應資料表，並排查刪除後殘留 metadata 導致的「Page不存在」「Action不存在」等錯誤"
version: 2
created: "2026-08-15"
updated: "2026-08-15"
---
## When to Use
當使用者要求在 AiPower（或其他基於 Quicksilver/Jeedsoft 框架的系統，特徵：TsMenu/TsUnit/TsPage/TsForm 等 Ts*/Tc*/Tw* 命名的資料表）精簡選單、移除模組、刪除資料表，或系統畫面跳出「ID為XXX的Page不存在」「不存在單元編碼為XXX的Action」等錯誤時使用。這類系統的選單、頁面、單元(Unit)、表單、權限都是資料驅動的 metadata，直接砍資料表會留下大量隱藏殘留引用。

## Procedure
1. 連線前先確認 MariaDB4j 內嵌資料庫的 port：ssh 到主機後執行 powershell -Command "(Get-CimInstance Win32_Process -Filter \"name='mariadbd.exe'\").CommandLine"，port 會隨 Tomcat 每次重啟而改變，不要沿用舊 port。
2. 刪除選單前，先查 TsMenuAd（closure table，記錄選單的祖先-子孫關係）取得完整子樹的所有 FId，同一併加上根節點本身，再用暫存表 DELETE FROM TsMenu WHERE FId IN (...)，同時清 TsRoleMenu/TsMenuNumber/TsWorkbench 等關聯。
3. 刪除選單後務必檢查孤兒：SELECT * FROM TsMenu m WHERE m.FParentId IS NOT NULL AND NOT EXISTS (SELECT 1 FROM TsMenu p WHERE p.FId=m.FParentId)，重複執行直到 0 筆（可能有多層鏈式孤兒）。
4. 若要砍實際資料表（DROP TABLE），先反查該表對應的 TsUnit（透過 TsPage.FUnitId -> TsUnit.FId，或直接 TsUnit.FTable=表名），確認這個 Unit 沒有被其他保留中的選單/頁面共用（join TsMenu/TsPage 檢查是否還有存活引用）。
5. DROP TABLE 前，先清除該 Unit 的全部 metadata：TsField/TsForm/TsFormField/TsList/TsListColor/TsEdit/TsQuerySchema/TsPrivilege/TsDataPrivilege/TsRoleUnitPrivilege/TsHomepageItem/TsCollection 等，最後才刪 TsUnit 本身與 DROP TABLE。
6. 全庫掃描孤兒引用時，不能只查 FUnitId 欄位，還要查所有變體命名：FUnitId1/FUnitId2(TsRelation)、FMasterUnitId、FEntityUnitId、FSlaveUnitId、FContextId、FEntityId 等——用 information_schema.COLUMNS 動態產生 UNION ALL 查詢批次執行(300筆一批避免逾時)，比對是否有值指向已刪除的 TsUnit.FId。
7. 同樣要掃描資料內容本身(不只是欄位級別引用)：例如 TcXxxCollectionSetup.FSql 這類存了整段 SQL 字串的欄位，可能寫死查詢已刪除的表並回傳已刪除 Unit 的 ID，用 WHERE column LIKE '%關鍵字%' 全表全欄位掃描找出。
8. 若清完資料庫後錯誤仍持續出現，先確認 Tomcat 是否已重啟(metadata 有記憶體快取)。重啟後若還是報錯，改查 Tomcat access log (localhost_access_log.*.txt) 搜尋錯誤訊息中的關鍵字(如 CallLog)，找出前端主動呼叫的 API 路徑(如 POST /aipower/Ecp.CallLog.getExtensionLength.data)，這代表是前端JS寫死呼叫，不是資料殘留問題。
9. 若確認是前端寫死呼叫某個已刪除 Unit 的 Action，且該 Action 的 Java class 仍存在於 webapps/WEB-INF/lib/*.jar 中(用 PowerShell + System.IO.Compression.ZipFile 掃描 jar 內容確認 class 是否存在)，最務實的解法是把該 Unit 定義補回 TsUnit(用同類型現存 Unit 當範本填 FHomeClassName/FDaoClassName/FServiceClassName/FActionClassName)，並視需要用 jar 內附的 QS-MODULE/data/sql/*.sql 建表腳本重建對應的空資料表，讓 Action 能被正確路由，而非硬要移除前端呼叫。
10. 重啟 Tomcat 的方式：因為 server.bat 用 catalina run 前景模式執行，SSH session 結束會連帶殺掉程序，須用 schtasks /Create /TN <任務名> /TR "cmd /c cd /d C:\Aipower && server.bat" /SC ONCE /ST 00:00 /F 建立排程工作，再用 schtasks /Run /TN <任務名> 觸發，這樣程序才能真正脫離 SSH session 背景執行。
11. 首頁若有「最近打開記錄」類小工具(對應 TsCollection 表)，清除模組後裡面會殘留指向已刪除實體的紀錄(即使不報錯也是髒資料)，可用 SELECT c.*,實體是否存在 的方式逐筆比對後清除失效項，或整批 TRUNCATE/DELETE FROM TsCollection 讓其重新累積。
12. 每一輪 DELETE/DROP 操作前都要先備份：用 mysqldump 指定明確的表名清單(不要用萬用字元)備份到獨立、確定不會被系統自動清理的路徑，例如另建一個與 C:\Aipower 主目錄平行的資料夾(如 C:\aipower_db_backup)，因為 C:\Aipower 目錄下的檔案可能被 Tomcat 啟動腳本(kill-leftover.ps1)或應用程式自動清除。

## Pitfalls
- 絕對不要用 del 或 rm 搭配萬用字元(如 *.sql)清理備份/暫存檔案，容易連同關鍵備份一起誤刪且無法復原——逐一列出明確檔名刪除，或存放備份時就跟臨時工作檔案放在不同目錄。
- MariaDB4j 內嵌服務的 port 每次 Tomcat 重啟都會改變(隨機分配)，長時間作業中途一定要重新確認 port，不能假設上一輪連線用的 port 還有效。
- 清除選單/資料時，資料庫可能存在歷史孤兒垃圾(FTreeSerial='isolated' 之類)，這些不是你這次操作造成的，遇到孤兒節點的父節點也查無資料時，先去查該父節點ID是否『從一開始就不存在於資料表』(用之前的完整備份比對)，避免誤判是自己的操作導致，同時可以借機一併清除這些歷史垃圾。
- 只清除選單(TsMenu)不等於清除功能——選單背後還有頁面(TsPage)、表單(TsForm)、查詢架構(TsQuerySchema)、權限(TsPrivilege)、以及實際業務資料表，四層是分開的，砍選單不會動到底層資料，若使用者要求『徹底清除』要主動確認清到哪一層。
- DROP TABLE 前一定要檢查該 Unit 對應的 TsUnit 是否被其他『保留中』的選單/頁面共用，直接用『看選單有沒有連到』來判斷是不夠的——底層 metadata(TsField/TsForm/TsHomepageItem/TsCollection/TsRelation等)可能有獨立於選單樹之外的引用，必須做全庫的欄位掃描才能確認乾淨。
- 系統框架常有『全域性』會在每次登入/換頁時查詢的核心表(如本例的 TsMarquee 跑馬燈、TsSystemMessage 系統消息)，即使它們掛在某個要被刪除的選單分類底下，也不代表可以隨意刪除其資料表——判斷標準應該是『這個資料表/Unit是否被系統框架級別的通用邏輯依賴』，不能只看選單分類位置。
- 清除孤兒引用時系統參數(TsSystemParameter)的值不能直接設成空字串，若該欄位原本存的是 UUID(如 QsSecondaryHomepage)，設空字串會讓程式解析 UUID 失敗拋出 'Invalid UUID string' 的新錯誤——要嘛保留原值只把 Enabled 開關關閉，要嘛改指向一個確實存在的有效實體。
- 看到系統報錯訊息包含某個已刪除模組的名稱(如 Ecp.CallLog)，不代表資料庫還有『資料層級』的殘留可清——有可能是前端 JS 檔案寫死呼叫該 API 端點，此時清資料庫是徒勞的，要去查 Tomcat 的 access log 找出實際被呼叫的 API 路徑來確認問題本質，而不是一直在資料庫裡瞎找。
- 備份檔案存放的目錄若跟被清理/被壓縮/被應用程式自動清除的目錄相同(例如系統程式安裝目錄)，即使沒有主動刪除，也可能被其他自動化流程(kill-leftover.ps1、應用啟動清理邏輯、使用者的其他操作)意外清空——重要備份要存到明確獨立、跟主程式無關的路徑。
- 已知案例(AiPower/艾波羅系統，可能是原廠安裝時就存在的破損 Unit，非本次操作造成)：TsUnit 裡存在 3 個『Unit 定義還在但實際資料表已不存在』的破損項目——Ecp.Contact(FTable=TcContact，只剩一個叫 tccolleague 的 VIEW)、Ecp.SkillSetup(FTable=TcSkillSetup)、Ecp.VRM(FTable=Record，注意表名就叫 Record 不是 TcXxx 慣例)。這三者原廠出貨時可能就是未授權/未啟用的模組，資料表從未真正建立。若這三者沒有被任何存活選單/頁面/首頁小工具引用(用第4/5類健康檢查確認 page_cnt/menu_cnt)，可以安全保留不動；若使用者要求徹底清乾淨，才需要連同 metadata 一併移除(參考步驟5的清單)。下次遇到同一套系統，可直接用這三個 FCode 查詢現況，不必重新掃描一次全庫。
- 『Ecp.Contact』與『Ecp.CallLog』是關聯度很高的一對——TcContactCollectionSetup.FSql 這個聯絡人匯總設定表，其 SQL 字串裡會直接查詢 TcCallLog 並在結果中寫死回傳 Ecp.CallLog 的 Unit ID(5d6b7749-a28f-496b-9eff-524cb20474fc 是本例的固定值，不同安裝可能不同，但『找法』通用)，這種『資料內容夾帶 Unit ID』的隱性依賴，用一般的欄位級 FUnitId 掃描抓不到，只能用 LIKE 全表全欄位掃描 Unit 的 FCode 字串或 FId 字串才找得到。刪除 CallLog 模組時務必連帶檢查並清理/處理 TcContactCollectionSetup。
## Verification
1. 執行 SELECT COUNT(*) FROM TsMenu m WHERE m.FParentId IS NOT NULL AND NOT EXISTS (SELECT 1 FROM TsMenu p WHERE p.FId=m.FParentId) 應為 0，確認選單樹無孤兒。
2. 執行 SELECT COUNT(*) FROM TsPage p WHERE p.FUnitId IS NOT NULL AND p.FUnitId<>'' AND NOT EXISTS (SELECT 1 FROM TsUnit u WHERE u.FId=p.FUnitId) 應為 0，確認頁面都有對應存活的 Unit。
3. 執行 SELECT COUNT(*) FROM TsUnit u WHERE u.FTable IS NOT NULL AND NOT EXISTS (SELECT 1 FROM information_schema.tables t WHERE t.table_schema=DATABASE() AND LOWER(t.table_name)=LOWER(u.FTable)) 應為 0，確認每個 Unit 定義都對應到實際存在的資料表。
4. 對所有含 FUnitId 或其變體欄位名的表做動態批次掃描，確認沒有任何一筆記錄指向已刪除的 TsUnit.FId。
5. 全庫用 LIKE '%關鍵字%' 掃描已刪除模組名稱(如表名、Unit編碼)是否還殘留在任何欄位內容中，特別留意存 SQL 字串的欄位(如 XxxCollectionSetup.FSql)。
6. 重啟 Tomcat 後，實際用瀏覽器登入操作一輪(或用 curl 直接打有問題的 API 端點觀察回應)，確認錯誤訊息不再出現、且不是被新的錯誤(如 Invalid UUID)取代。
7. 查看 Tomcat 的 localhost_access_log 確認該錯誤相關的 API 呼叫回應碼與內容是否恢復正常(例如從『Action不存在』變成正常回應或至少變成『未登入』這類預期內錯誤)。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
