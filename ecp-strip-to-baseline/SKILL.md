---
name: ecp-strip-to-baseline
description: Strip a Chainsea ECP / aipower (Quicksilver-based) instance down to a bare Qs.* framework skeleton — remove all business CRM/workflow/custom modules while keeping the platform usable — then reactively un-break whatever the base UI shell turns out to hard-depend on. Use when the user wants to "clean out all features and build from scratch" on an ECP/aipower/Quicksilver deployment, or is chasing "編碼為 X 的單元/頁面不存在" / "不存在單元編碼為 X 的 Action" errors after such a cleanup.
---

# ECP/aipower：清空業務功能只留框架骨架，並收拾殘局

一次完整跑過的方法論：把一套 Chainsea ECP / aipower（Quicksilver 框架）資料庫清到只剩平台骨架，
再處理「殼層其實偷偷依賴某幾個業務模組」的連鎖反應。首次實戰見 2026-07-13
`aipower-docker-local`（`C:\Users\HCH\aipower`，port 22821）。方法論本身不綁定特定實例，換一套
Quicksilver 系統（Lab2/ECP AI3 等）一樣適用，只要換連線參數。

## 核心概念：Unit 命名空間怎麼分

`TsUnit.FCode` 前綴決定了這個 Unit 是「平台骨架」還是「業務功能」：

| 前綴 | 意思 | 能不能清 |
|---|---|---|
| `Qs.*` | Quicksilver 框架自我描述的 metadata（選單、頁面、欄位、權限本身怎麼運作）| **絕對不能清** — 這是低代碼平台的 schema catalog，清了平台直接壞掉 |
| `Wf.*` | 工作流引擎（Workflow/Process/WorkItem/Activity/Node/Line/Event/Button/Version）| **不能清**——實測發現首頁「快捷工作檯」寫死用 ID 直接查 `Wf.WorkItem`，這是平台核心基建，不是業務功能 |
| `Ecp.*` | CRM/客服業務模組（聯絡人、企業、服務請求、聊天...）| 大部分可以清，但有一小撮是殼層 chrome 寫死依賴（見下方清單）|
| `Aipower.*` | 廠商自己疊加的加值模組（AIFF、點數活動、服務咨詢...）| 可以清 |

## 已知「看起來像業務功能、其實是殼層 chrome 寫死依賴」的 Unit

這些即使命名是 `Ecp.*`，一旦刪除就會讓**所有帳號**在首頁/工具列直接跳錯誤視窗（不是點進特定業務
頁面才會炸，是每次登入都會炸），必須跟著 `Qs.*`/`Wf.*` 一起留著：

| Unit | 中文名 | 為什麼是 chrome | 症狀 |
|---|---|---|---|
| `Wf.*`（全部 9 個）| 工作流引擎 | 首頁「快捷工作檯」用固定 ID `00000000-0000-0000-0001-000000003005` 直查 `Wf.WorkItem` | 首頁跳「ID為...的Unit不存在」|
| `Ecp.ChatAgent` | 聊天值機人員 | 頂部工具列的客服就緒/未就緒狀態列，每次登入都呼叫 | 「不存在單元編碼為Ecp.ChatAgent的Action」|
| `Ecp.CallLog` | 通話記錄 | 頂部工具列的來電彈窗監聽 | 「不存在單元編碼為Ecp.CallLog的Action」|
| `Ecp.ChatWorkGroup` | 文字客服群組 | `Ecp.ChatAgent` 內部查詢群組成員時的直接依賴 | ChatAgent 狀態查詢丟 NullPointerException（見下方「已知良性殘留」，這顆其實是原始資料本來就有的 bug，跟清除無關）|
| `Ecp.DataTransfer` | 業務資料轉移 | 工具列權限檢查固定呼叫 | 「Cannot invoke ...DataTransferHome.getService() is null」|
| `Ecp.HelpDesk` | 服務台 | 頂部「幫助」選單固定連結 | 「編碼為Ecp.HelpDesk的單元不存在」——**但這個連結本身編譯寫死在 `aipower-module-crm-*.jar` 的 Java bytecode 裡，資料庫/JS/JSP 都找不到，無法乾淨移除，只能留著或接受它報錯**（見下方限制說明）|
| `Ecp.Aile` | Aile與MECP資料同步 | 某些部署把網站根目錄 `/aipower/`（不帶頁面代碼時）直接指向 `Ecp.Aile.Login`（一個訪客登記表單，FTitle「Aiff登錄」），不是標準員工登入頁 | 根目錄跳「編碼為Ecp.Aile.Login的頁面不存在」；就算補回來，這個表單本身可能卡在載入中空白——**改連 `Qs.OnlineUser.Login.page` 走標準登入即可繞開**，不必修這個表單 |

`Ecp.ChatWorkGroup` 這一顆比較特殊：它連著更深的 CRM 概念（`Ecp.Contact` 聯絡人），如果要讓
「服務台」新增表單裡的聯絡人挑選欄位也正常運作，得把 `Ecp.Contact` 也救回來——但 Contact 是 CRM
的骨幹，救回它大概率會連帶讓其他業務模組（服務請求、企業客戶...）因為同樣缺 Contact 而繼續報錯。
**遇到這種「往下挖會牽動真正 CRM 核心」的分岔點，停下來問使用者要不要繼續深挖，不要自己決定。**

## 已知良性殘留（不用管，不是清除造成的）

- `TcChatWorkGroupUser`（客服人員群組歸屬）從**原始備份**開始就是 0 筆——`Ecp.ChatAgent` 的狀態
  查詢對著 0 筆資料會丟 `Cannot invoke List.isEmpty() because "groups" is null`，這是廠商程式碼
  本來就有的 null 判斷 bug，跟這次清除工程無關，清除前就會炸。
- 選單樹裡可能存在 `FPageId IS NULL` 的孤兒節點（例如「工時日誌」）——這種節點本來就沒有掛實際
  頁面，點下去原本就無作用，發現了就直接 `DELETE FROM tsmenu WHERE FId=...` 清掉，不用去救任何
  Unit。

## 步驟

### 1. 清除前：一定要有完整備份

```bash
docker exec <mariadb_container> mariadb -u root -p<pass> <db> > backup_$(date +%Y%m%d_%H%M%S).sql
```

沒有這個，下面所有操作都不該做。`aipower-docker-local` skill 有現成的
`backup-restore.ps1`（互動選單）可以用。

### 2. 清除業務模組

```bash
scripts/strip_units.sh <container> <db_user> <db_pass> <db_name> 'Ecp.%' 'Aipower.%' 'AiPower.%' 'Wf.%'
```

**注意**：上面這行故意示範「連 Wf.\* 都清掉」的錯誤示範——已知結論是 **Wf.\* 不該清**，正確的第一刀
應該是：

```bash
scripts/strip_units.sh <container> <db_user> <db_pass> <db_name> 'Ecp.%' 'Aipower.%' 'AiPower.%'
```

腳本原理：動態抓出所有有 `FUnitId`/`FPageId` 欄位的表（跳過 `~` 開頭的多語系 VIEW 和
`tsunit`/`tspage`/`tsmenu` 本身），照 `Function型選單捷徑 → FPageId選單項 → 其他FPageId表 →
其他FUnitId表 → tspage本體 → tsunit本體 → 反覆收斂空的Directory選單節點` 的順序刪，整包包在單一
交易裡，任何一步出錯就整批 rollback（mariadb client 沒帶 `--force`，中途出錯連線關閉=隱含
rollback，不會留下半殘狀態）。

**重要限制**：這套資料庫沒有真正的 InnoDB 外鍵約束（`information_schema.KEY_COLUMN_USAGE` 查
`REFERENCED_TABLE_NAME IN ('tsunit','tspage')` 是空的），referential integrity 全部靠 Java 應用層
自己維護。這代表：
- 刪除順序理論上不受 DB 擋，但邏輯上還是要照上面順序做，因為子查詢（`SELECT FId FROM tsunit
  WHERE ...`）要在 `tsunit`/`tspage` 本體還沒被刪之前執行才拿得到正確的 ID 集合。
- 有些關聯不是走 `FUnitId`（例如 `TsPage.FMasterUnitId` 用在「slave 頁面」，`TsMenu.FCountUnitId`
  用在選單角標計數）——這些欄位目前腳本沒有涵蓋，代表某些 slave/count 相關的殘留列可能不會被清到。
  實測發現這反而無害（該列本來就是通用模板渲染，不會因為主 Unit 沒了而報錯），但如果之後遇到新的
  怪錯誤，先查這兩個欄位。

### 3. 重啟 + 抽測

```bash
scripts/smoke_test.sh <app_container> http://localhost:<port>/aipower administrator <pwd>
```

Tomcat 會快取 Unit/Page/Menu metadata，**每次改完資料庫都要重啟 app 容器**，不然看到的還是舊狀態。

### 4. 密碼卡住怎麼辦

清除過程如果要頻繁試登入，別跟系統的登入失敗鎖定機制對賭，直接改資料庫：

```bash
scripts/reset_admin_password.sh <db_container> <db_user> <db_pass> <db_name> administrator 111111 <python_bin>
```

Windows 上如果 `python3` 指向 Microsoft Store 空殼，找真正的直譯器（`/d/python3.13/bin/python`
之類），當作最後一個參數傳進去。

### 5. 使用者實際操作時回報「XX 不存在」→ 反應式抓漏迴圈

1. 先判斷：這是「Qs.\*/Wf.\* 或已知 chrome 清單」裡的東西，還是貨真價實的業務功能？
   - 是前者 → 直接進第 2 步救回來。
   - 是後者但只是「殘留指標指向已經正確刪除的東西」（例如系統參數 `QsSecondaryHomepage`、
     `tsparameterdefinition.FEntityUnitId`）→ 不要救單元，**清掉那個殘留指標本身**（見下方
     `QsSecondaryHomepage`/`AipowerDefaultProduct` 案例）。
   - 是後者且會繼續往下牽動更核心的 CRM 概念（例如服務台需要聯絡人）→ **停下來問使用者**要繼續
     深挖還是放著。
2. 找出缺的是哪個 Unit/Page ID：
   ```bash
   grep -o "INSERT INTO \`tsunit\` VALUES ('[^']*', 'Ecp.XXX'[^;]*;" default.sql
   ```
3. 精準救回（不影響其他已清除的東西）：
   ```bash
   scripts/restore_units.sh <db_container> <db_user> <db_pass> <db_name> default.sql <unit_id...>
   ```
4. 重啟 + 抽測（回到步驟 3）。

### 6. 「找不到是哪裡觸發的」怎麼辦

錯誤有時候不是選單/系統參數直接指到的，而是某個 Unit 本身的 metadata 都齊全、卻在點某個具體按鈕
時才報錯（例如「幫助」選單裡的固定連結）。用這隻腳本查是資料驅動還是寫死在 Java bytecode 裡：

```bash
scripts/find_reference.sh <app_container> Ecp.HelpDesk
```

依序查 `quicksilver/`（框架殼層）→ `ecp/`（客製資源）→ 全站 JSP → `WEB-INF/lib/*.jar`。**如果只在
jar 裡找到，代表這是編譯進 Java bytecode 的固定選單項，資料庫/前端資源都改不掉，要嘛留著、要嘛接受
它報錯、要嘛走反編譯改 class 重新打包這種量級完全不同的工程（不建議，除非使用者明確要）。**

## 殘留指標清理案例（清指標本身，不是救單元）

**系統參數指向已刪除頁面**：

```sql
SELECT * FROM tssystemparameter WHERE FValue LIKE '%<被刪除的頁面ID開頭>%';
-- 找到後：
DELETE FROM tssystemparameter WHERE FId='<那一筆的FId>';
```

實測案例：`QsSecondaryHomepage`（第二主頁）系統參數一開始有**兩筆同 Key 的殘留資料**（不確定是原廠
demo seed 的既有瑕疵還是別的原因），一筆指向合法的 `Qs.Demo.SecondaryHomepage`，一筆指向已清除的
`Aipower.ServiceConsult.Main`。刪掉後者後錯誤一度消失，但**之後又復發**——變成只剩一筆、`FId` 換了
新的、但值又是那個壞頁面（意味著這個 Key 在某個時機點會被整組重寫/重新產生一個新 FId，具體觸發條件
未查清楚，懷疑跟開啟「系統參數」頁或存檔某個同群組參數有關，不影響下面的修法）。

**排錯過程中一個重要的死路**：直覺會覺得「錯誤跟這個功能開關有關」，於是嘗試
`UPDATE tssystemparameter SET FValue='0' WHERE FKey='QsSecondaryHomepageEnabled'`（關掉第二主頁功能）
——**這樣做完全沒用，重啟後錯誤照樣跳**。這證實了這個頁面查詢動作**不受 Enabled 開關保護**，不管
功能有沒有啟用，只要 `QsSecondaryHomepage` 這個值指向一個不存在的頁面，系統某處就是會去解析它並報
錯（很可能是用來決定按鈕圖示/文字要不要顯示之類的預先查詢，不是真的要開啟該分頁才查）。也試過清瀏覽
器 `localStorage.clear(); sessionStorage.clear()`，一樣沒用，排除是前端快取「上次開啟分頁」的可能。

**真正有效的修法**：不要刪除這個 Key、也不要關開關，直接把**值本身**改指向一個真實存在的頁面：

```sql
UPDATE tssystemparameter SET FValue='<一個真的存在的頁面ID，例如 Qs.Demo.SecondaryHomepage 的 FId>'
WHERE FKey='QsSecondaryHomepage';
```

改完值之後開關保持原本的 `1`（啟用）也完全沒問題——問題從來不是「要不要啟用」，而是「指向的目標存不
存在」。這個案例的教訓：**遇到「系統設定指向已刪除項目」的錯誤，優先找一個合法的替代目標把值改過去，
而不是急著關功能開關或刪設定列——開關/刪除都可能治標不治本，甚至完全沒用。**

**參數定義本身指向已刪除單元的 EntityBox 挑選器**：

```sql
SELECT pd.FName, pd.FCode, pd.FEntityUnitId
FROM tsparameterdefinition pd LEFT JOIN tsunit u ON pd.FEntityUnitId = u.FId
WHERE pd.FEntityUnitId IS NOT NULL AND u.FId IS NULL;
-- 找到後直接砍掉這筆參數定義（連同對應的 tssystemparameter/tsuserparameter 值）：
DELETE FROM tsparameterdefinition WHERE FCode='<那個FCode>';
DELETE FROM tssystemparameter WHERE FKey='<那個FCode>';
DELETE FROM tsuserparameter WHERE FKey='<那個FCode>';
```

實測案例：「系統參數」頁本身（`Qs.Parameter.System.page`）因為要列出全部參數定義，其中
「預設產品」（`AipowerDefaultProduct`）的 `FEntityUnitId` 指向已刪除的 `Ecp.Product`，整頁直接
炸開「ID為...的Unit不存在」。刪掉這筆參數定義後页面恢復正常，不需要救回 `Ecp.Product`。

**孤兒選單節點**（`FPageId IS NULL`，清除前就已經是死的）：

```sql
SELECT FId, FName, FPageId FROM tsmenu WHERE FName='<可疑的選單項>';
-- 確認 FPageId 真的是 NULL 之後：
DELETE FROM tsmenu WHERE FId='<那筆的FId>';
-- 順手檢查父節點是不是也變空了：
SELECT FName FROM tsmenu WHERE FParentId=(SELECT FParentId FROM tsmenu WHERE FId='<剛刪的那筆>');
```

## 相關 Skill

- `aipower-docker-local`：這套本機 Docker 實例本身的啟動/備份/還原（`backup-restore.ps1`）。
- `ecp-pwd`：**原生** mariadbd.exe 行程（`server.bat`/`database.bat` 這種安裝方式，非 Docker）的
  密碼重設腳本，跟這裡的 `scripts/reset_admin_password.sh`（Docker 專用）是同一套演算法、不同連線方式。
- `ecp-cross-version-jar-porting`：如果真的要動 `WEB-INF/lib/*.jar` 裡編譯好的 class（例如
  `Ecp.HelpDesk` 那種寫死選單項），這隻 skill 有完整的反編譯/替換方法論。
- `ecp-schema` / `ecp` / `ecp-java-core`：更廣泛的 ECP/Quicksilver 資料庫結構與分層架構知識庫。

---

## Conformance Addendum

## When to Use
Strip a Chainsea ECP / aipower (Quicksilver-based) instance down to a bare Qs.* framework skeleton — remove all business CRM/workflow/custom modules while keeping the platform usable — then reactively un-break whatever the base UI shell turns out to hard-depend on. Use when the user wants to "clean out all features and build from scratch" on an ECP/aipower/Quicksilver deployment, or is chasing "編碼為 X 的單元/頁面不存在" / "不存在單元編碼為 X 的 Action" errors after such a cleanup.

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
