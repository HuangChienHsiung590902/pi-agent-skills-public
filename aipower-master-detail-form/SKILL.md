---
name: aipower-master-detail-form
description: 在 aipower/ECP（Quicksilver 框架）建立母子表單（一對多，類似 Access「主表單含子表單」）的兩種原生做法——(1) 頁籤式：TsRelation + 從屬 TsPage（FMasterUnitId+FIsSlavePage），點頁籤才切換，適合「客戶→多筆訂單」這種鬆散關聯；(2) 上下堆疊式：TsUnit 本身的 FIsSlaveUnit+FMasterUnitId+FMasterField（真正的「從屬單元」），母表單同一頁下方自動內嵌可編輯明細表格（新增/刪除/上升/下降/置頂/置底全部框架自動產生），適合「訂單→多筆明細」「費用→多筆費用明細」這種緊密的一對多，完全比照系統內建 Ecp.Order/OrderDetail、Ecp.Expense/ExpenseDetail 反查得出。含完整可複製的 SQL 範本、六大核心 Java 樣板、以及一系列反查/實測踩過的坑（TsUnit.FIcon 快取、母表單頁籤的 FUnitId 故意設成子單元 ID 這個違反直覺的關鍵、InputBox-Number 不存在、TsRoleMenu 選單授權、embedded MariaDB port 每次重啟會變）。當使用者要求「做一個母子表單」「一對多表單」「像 Access 那種 subform」「訂單/費用 那種上下佈局的表單」「一頁同時看到主表跟明細」時使用。
---

# aipower 母子表單（一對多）建立指南

在 aipower（本機 `C:\Aipower`，Quicksilver 框架）建立「母子表單」有**兩種完全不同、互不相容的原生機制**，選錯會做出「看起來像但行為不對」的結果（例如使用者要「同頁上下堆疊」，卻做成「要點頁籤才切換」）。**動手前務必先問清楚使用者要哪一種**，兩者外觀差異很大：

| | 頁籤式（Tab） | 上下堆疊式（Stacked，真正的「從屬單元」） |
|---|---|---|
| 使用者怎麼看到子資料 | 開啟母記錄表單，點左側「訂單」等頁籤才切換顯示 | 開啟母記錄表單同一頁，欄位群組下方就是完整寬度的明細表格，不用點擊 |
| 系統內建範例 | `Ecp.Customer.OrderList`（客戶表單裡的「訂單」頁籤）、`Ecp.Task.ExpenseList` 等 | `Ecp.Order`+`Ecp.OrderDetail`、`Ecp.Expense`+`Ecp.ExpenseDetail`、`Ecp.Contract`+`Ecp.ContractDetail` |
| 底層機制 | `TsPage.FMasterUnitId` + `FRelationId`（`TsRelation`）+ `FIsSlavePage=1`，額外建一頁 `FType='EntityList'` 當頁籤 | `TsUnit.FIsSlaveUnit=1` + `FMasterUnitId` + `FMasterField`（單元本身的屬性，不是頁面屬性） |
| 明細表格工具列 | 要自己在 `TsToolItem` 設定新增/打開/刪除/重新整理 | **完全不用設定**，框架看到 `FIsSlaveUnit` 自動產生新增/刪除/上升/下降/置頂/置底 |
| 子單元子表需要排序欄位嗎 | 不需要 | 建議加 `FIndex int(11)` 欄位，上升/下降/置頂/置底才有東西可以動 |
| 適合場景 | 關聯鬆散、子項目本身也常被其他母單元引用（如訂單同時掛在客戶/合約/報價單底下） | 子項目就是母記錄不可分割的一部分（訂單明細離開訂單沒有意義） |

**這份 skill 兩種都收錄**，因為使用者一開始以為要的是 A，實際上（比照 Access 的 Orders/Order Details 範例、或系統內建「費用」表單）真正想要的是 B——**遇到「像 Access 那種 subform」「上下佈局」「一頁看到全部」的描述，優先假設是 B（上下堆疊/從屬單元），不要預設走 A（頁籤）**。

## 環境前提（每次動手前先確認）

- 目標實例：`C:\Aipower`（本機，port 22821/22822，內嵌 MariaDB4j）。
- **內嵌 MariaDB 的 port 每次 `server.bat` 重啟都會換**，不要沿用上次的 port 號，每次都要重新查：
  ```powershell
  (Get-CimInstance Win32_Process -Filter "Name='mariadbd.exe'").CommandLine
  # 從輸出裡找 --port=xxxxx
  ```
- 重啟流程（`TsUnit`/`TsPage`/`TsToolItem`/`TsRoleMenu` 這類 metadata 全部要重啟才會被 Registry/前端快取讀到新值，改完 DB 沒重啟＝白改）：
  ```powershell
  powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Aipower\kill-leftover.ps1"
  Start-Process -FilePath "C:\Aipower\server.bat" -WorkingDirectory "C:\Aipower" -WindowStyle Hidden
  ```
  然後 `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:22821/aipower/` 輪詢到 200。
- **驗證優先用 curl 直接打 API，不要依賴瀏覽器**——這台機器上 Playwright/CDP 接管的 Chrome 常常跟使用者本人同時在用（共用同一個瀏覽器），會互搶、彈出一堆 `beforeunload` 對話框、甚至連線斷掉。登入换 cookie 後直接測 CRUD：
  ```bash
  curl -s -c cookie.txt -X POST "http://127.0.0.1:22821/aipower/Qs.OnlineUser.login.data" \
    -H "Content-Type: application/json" \
    -d '{"loginName":"administrator","password":"<密碼>","language":"zh-tw"}'
  curl -s -b cookie.txt -X POST "http://127.0.0.1:22821/aipower/Ecp.Xxx.save.data" \
    -H "Content-Type: application/json" -d '{"data":[{...}]}'
  curl -s -b cookie.txt -X POST "http://127.0.0.1:22821/aipower/Ecp.Xxx.getListData.data" \
    -H "Content-Type: application/json" -d '{"pageSize":10,"pageIndex":1}'
  ```
  **⚠ 絕對不要重設/修改 `administrator` 密碼**——即使懷疑登入失敗也不行，先跟使用者要目前密碼。

## 六大核心 Java 樣板（兩種模式共用）

母子表單的母/子單元都各自是一個標準六大核心 Unit（Model/Home/Dao/DaoImpl/Service/ServiceImpl/Action/ActionImpl，8 個檔案），寫法完全一致：

```java
// Model：EntityModel 子類別，四個建構子＋每個業務欄位的 getter/setter
public class XxxModel extends EntityModel {
    private static final long serialVersionUID = 1L;
    public XxxModel() {}
    public XxxModel(Record record) { super(record); }
    public XxxModel(ResultSet rs) { super(rs); }
    public XxxModel(JSONObject json) { super(json); }
    public String getFName() { return getString("FName"); }
    public void setFName(String value) { put("FName", value); }
    // 數值/日期型別對照 EntityModel 繼承鏈（BaseModel extends Record）的 getter：
    // 文字 getString / 整數 int getInt（沒有 getInteger！）/ 金額 BigDecimal getDecimal（沒有 getBigDecimal！）
    // 日期 Date getDate / UUID getUuid
}

// Home：靜態 UNIT_ID + Registry 存取
public class XxxHome {
    public static final UUID UNIT_ID = UUID.fromString("<在此貼 UUID>");
    public static XxxDao getDao() { return (XxxDao) Registry.getDao(UNIT_ID); }
    public static XxxService getService() { return (XxxService) Registry.getService(UNIT_ID); }
}

// Dao interface：extends EntityDao<XxxModel> {}
// DaoImpl：extends EntityDaoImpl<XxxModel> implements XxxDao {
//     public XxxDaoImpl() { super(XxxHome.UNIT_ID, XxxModel.class); }   // ⚠ 這個建構子絕對不能省略！
// }
// Service/ServiceImpl、Action/ActionImpl 完全同構，建構子都要呼叫 super(UNIT_ID, XxxModel.class)
```

**⚠ 致命坑**：Dao/Service/Action 的 Impl 若沒寫這個明確呼叫 `super(UNIT_ID, XxxModel.class)` 的建構子（例如寫成完全空的 `{}`），`EntityDaoImpl`/`EntityServiceImpl`/`EntityActionImpl` 的無參數建構子會把內部 `modelClass` 寫死成基底 `EntityModel.class`（不是 `null`），導致框架永遠不會用反射自動推導真正子類別，一旦呼叫 `getItem`/`update` 之類需要轉型的方法就會 `ClassCastException`。三層都要寫，缺哪層就是哪層的操作會炸。

編譯部署：
```bash
mkdir -p /tmp/classes
cd /c/Aipower/apache-tomcat/webapps/aipower/WEB-INF/lib
CP=$(cygpath -w "$(pwd)")"\\*"
/c/Aipower/jdk/bin/javac.exe -cp "$CP" -d "$(cygpath -w /tmp/classes)" <逐一列出所有 .java 檔的 cygpath -w>
# 全部 class 檔複製進 WEB-INF/classes 對應 package 路徑，全新 package 記得 mkdir -p
```

---

## 模式 A：頁籤式（TsRelation + 從屬 TsPage）

適合：母記錄開啟後，子清單以「額外一個頁籤」呈現，點了才看到。

### SQL 骨架

```sql
-- 母單元 TsUnit（正常單元，不需要任何 slave 標記）
INSERT INTO TsUnit (FId,FCode,FName,FIcon,FEditId,FModuleId,FOpenMode,FIsSlaveUnit,FDataSource,FTable,
  FKeyField,FKeyType,FNameField,FDescription,FHomeClassName,FDaoClassName,FServiceClassName,FActionClassName,FApiClassName)
 VALUES (@M_UNIT,'Ecp.Xxx','...',...,'System',b'0',...);

-- 子單元 TsUnit（一樣是正常單元，這個模式子單元本身不需要 FIsSlaveUnit）
INSERT INTO TsUnit (...) VALUES (@D_UNIT,'Ecp.XxxDetail',...);

-- 母單元自己的 List/Form（標準）
INSERT INTO TsPage (...) VALUES (@M_PLIST,...,'Ecp.Xxx.List','EntityList',@M_UNIT,NULL,b'0',...);
INSERT INTO TsPage (...) VALUES (@M_PFORM,...,'Ecp.Xxx.Form','EntityForm',@M_UNIT,@M_UNIT,b'1',1,...);
--                                                                        ^FUnitId=自己  ^FMasterUnitId=自己(自我參照,第一個頁籤)

-- 子單元「當作頁籤」的那一頁：FUnitId=子單元自己，FMasterUnitId=母單元，
-- FRelationId=下面建的關聯，FIsSlavePage=1，FIndex=2（排在母表單頁籤之後）
INSERT INTO TsPage (FId,FName,FTitle,FCode,...,FUnitId,FMasterUnitId,FRelationId,FIsSlavePage,FIndex,...)
 VALUES (@D_TAB_PAGE,'...','子項目','Ecp.Xxx.DetailTab',...,@D_UNIT,@M_UNIT,@REL_M_D,b'1',2,...);

-- TsRelation：兩個方向都要自己插入！只插單向會讓 Tomcat 啟動時框架自帶的
-- complementRelations() 嘗試自動補反向，但該常式沒帶 FId，直接 NOT NULL 炸掉整個啟動。
INSERT INTO TsRelation (FId,FOppositeId,FName,FOppositeName,FUnitId1,FUnitId2,FType,FField1,FField2,FDeleteAction1,FDeleteAction2)
 VALUES
 (@REL_M_D,@REL_D_M,'母-子','子-母',@M_UNIT,@D_UNIT,'field','FMasterId','FId','cancel','unset'),
 (@REL_D_M,@REL_M_D,'子-母','母-子',@D_UNIT,@M_UNIT,'field','FId','FMasterId','unset','cancel');

-- 子單元隱藏的 FK 欄位（不進清單/表單顯示，純供關聯用）
INSERT INTO TsField (FId,FUnitId,FName,FTitle,FType,...,FVisible,...,FEntityUnitId,...)
 VALUES (@D_F_MASTERID,@D_UNIT,'FMasterId','母記錄','EntityBox',...,b'0',...,@M_UNIT,...);

-- 頁籤頁的 TsToolItem 要自己配（新增/打開/刪除/重新整理），這個模式不會自動產生
INSERT INTO TsToolItem (...) VALUES (...,@D_TAB_PAGE,'Add',...,'EntityList.doAdd'),(...,'Refresh',...,'EntityList.doRefresh');
```

### 額外：清單頁「切換窗格」（選一列即時看子資料，不用打開整個表單）

```sql
UPDATE TsPage SET FHasViewFrame=b'1' WHERE FId=@M_PLIST;  -- 母清單頁開啟右側閱覽窗格
-- 另外建一個 FType='EntityView' 的頁面 + TsViewItem（FieldGroup 顯示母欄位 + List 顯示子清單）
INSERT INTO TsPage (...) VALUES (@M_VIEW,...,'Ecp.Xxx.View','EntityView',@M_UNIT,NULL,...);
INSERT INTO TsViewItem (FId,FName,FIndex,FUnitId,FType,FTitleType,FAlign,FVisible)
 VALUES (@VI_FIELDGROUP,'基本資訊',1,@M_UNIT,'FieldGroup','None','top',b'1');
INSERT INTO TsViewItemField (FId,FFieldName,FViewItemId,FIndex) VALUES (...);
INSERT INTO TsViewItem (FId,FName,FIndex,FUnitId,FType,FTitleType,FTitleText,FRelationId,FRecordCount,FAlign,FVisible)
 VALUES (@VI_LIST,'子項目',2,@M_UNIT,'List','Text','子項目',@REL_M_D,10,'top',b'1');
```

### 完整可複製範本

`C:\Aipower\tool\src\democustomerorder_unit.sql`（客戶/訂單，頁籤式）+
`C:\Aipower\tool\src\democustomerorder_viewframe.sql`（同上，加切換窗格）——
兩份都已在這台機器實測跑過、驗證過畫面，直接複製改名字/UUID/欄位即可。

---

## 模式 B：上下堆疊式（真正的「從屬單元」，優先假設使用者要這個）

適合：母記錄表單一開，欄位群組下方就是完整寬度的可編輯明細表格，**不需要點任何頁籤或按鈕**。這是反查系統內建 `Ecp.Order`/`Ecp.OrderDetail`、`Ecp.Expense`/`Ecp.ExpenseDetail` 才挖出來的機制，一開始會被 SQL 資料的表面樣子誤導，務必照下面的順序理解。

### 核心概念（違反直覺、務必先搞懂再動手）

1. **子單元的 `TsUnit` 本身要設三個欄位**：`FIsSlaveUnit=b'1'`、`FMasterUnitId=<母單元ID>`、`FMasterField='<子表FK欄位名>'`。這是**單元層級**的屬性，不是頁面層級——這才是框架判斷「這是從屬單元，要自動內嵌」的依據。
2. **母單元自己的「表單」那個 `TsPage` row，`FUnitId` 故意設成「子（從屬）單元」的 ID，不是母單元自己！** `FMasterUnitId` 才是設成母單元自己（自我參照）。也就是：
   ```sql
   -- Ecp.Xxx.Form（母單元的主表單頁，左側第一個頁籤，FTitle 固定顯示"表單"）
   INSERT INTO TsPage (FId,FCode,...,FUnitId,FMasterUnitId,FRelationId,FIsSlavePage,FIndex,...)
    VALUES (@M_PFORM,'Ecp.Xxx.Form',...,@D_UNIT,@M_UNIT,@REL_M_D,b'1',1,...);
   --                                    ^FUnitId=子單元ID  ^FMasterUnitId=母單元自己
   ```
   這是這個機制裡最容易搞反的一步——**兩種模式的 `Ecp.Xxx.Form` 頁面 FUnitId 語意完全不同**：模式 A 是「自己(母)」，模式 B 是「子(從屬單元)」。
3. **母單元自己的欄位群組內容**，不是來自這個 `TsPage`，而是來自一筆獨立的 `TsForm`（`FUnitId=母單元自己`，`FDefault=1`，`FPageId=NULL`）：
   ```sql
   INSERT INTO TsForm (FId,FName,FIndex,FUnitId,FPageId,FPlatform,FDefault,FGroupMode)
    VALUES (@M_FORM,'預設',1,@M_UNIT,NULL,'Computer',b'1','Double');
   -- 底下正常掛 TsFieldGroup/TsFormField
   ```
4. **子單元自己也要有一個「自我參照」的 `Form` 頁面**（`FUnitId=子單元自己`，`FMasterUnitId=子單元自己`，無 `FRelationId`）——這是點內嵌明細表格的「新增」或打開某一列時彈出的小表單，只有明細本身的欄位：
   ```sql
   INSERT INTO TsPage (FId,FCode,...,FUnitId,FMasterUnitId,FIsSlavePage,FIndex,...)
    VALUES (@D_PFORM,'Ecp.XxxDetail.Form',...,@D_UNIT,@D_UNIT,b'1',1,...);
   ```
   子單元自己的欄位一樣走獨立 `TsForm`（`FUnitId=子單元自己`，`FDefault=1`，`FPageId=NULL`）。
5. **內嵌明細表格的欄位設定，來自子單元自己的「預設」`TsList`**（`FDefault=1`，`FPageId=NULL`，不綁定任何特定 `TsPage`）：
   ```sql
   INSERT INTO TsList (FId,FUnitId,FName,FPlatform,FDefault,FMultiPage,FPageId,FIndex)
    VALUES (@D_LIST,@D_UNIT,'...清單（預設）','Computer',b'1',b'1',NULL,NULL);
   -- 底下正常掛 TsListField，欄位不含母子關聯用的 FK 欄位（那個要隱藏）
   ```
6. **⚠ 完全不需要（也不應該）為內嵌表格設定 `TsToolItem`**——新增/刪除/上升/下降/置頂/置底六顆按鈕是框架看到 `FIsSlaveUnit` 後自動產生的，反查 `Ecp.Order.Form`/`Ecp.Expense.Form` 的 `TsToolItem` 資料庫記錄，完全找不到這幾顆按鈕，只有 `Save`（保存）是手動配置的。
7. **子表的實體表建議加 `FIndex int(11)` 欄位**（可以是 NULL），對應上升/下降/置頂/置底排序，比照 `TcOrderDetail` 真實 schema。
8. **母子雙向 `TsRelation` 一樣要建**（`FField1`/`FField2` 對應 `TsUnit.FMasterField`），且一樣要兩個方向都自己插入，理由同模式 A 的踩坑說明（`complementRelations()` 崩潰 bug）。

### 反查驗證用的查詢（下次要找其他真實範例直接照抄）

```sql
-- 列出系統裡所有真正的「從屬單元」，挑一個跟你需求類似的當範本（Order/Detail 這種
-- header+line-items 最典型）
SELECT FCode,FIsSlaveUnit+0,FMasterUnitId,FMasterField FROM TsUnit WHERE FIsSlaveUnit=1;

-- 挑到目標後，反查它的 Form/List 頁面完整結構
SELECT FCode,FTitle,FUnitId,FMasterUnitId,FRelationId,FIsSlavePage+0,FIndex FROM TsPage
 WHERE FUnitId IN ('<母ID>','<子ID>') AND FPlatform='Computer';
SELECT FId,FName,FUnitId,FPageId,FDefault+0 FROM TsForm WHERE FUnitId IN ('<母ID>','<子ID>');
SELECT FId,FUnitId,FName,FDefault+0,FPageId FROM TsList WHERE FUnitId='<子ID>';
SELECT FCode,FDefaultEventHandler FROM TsToolItem WHERE FPageId='<母表單頁ID>';  -- 確認真的沒有明細按鈕
```

### 完整可複製範本

`C:\Aipower\tool\src\demoorder2_stacked_unit.sql`——`Ecp.DemoOrder2`（母，示範訂單）
+ `Ecp.DemoOrderLine`（子/從屬單元，示範訂單明細），已在這台機器實測跑過、透過
curl API（save.data/getListData.data）驗證母子資料正確建立與關聯，直接複製改
名字/UUID/欄位即可。

---

## 兩種模式共通的踩坑

- **`InputBox-Number` 不是合法的 `TsField.FType`**（這是把 CSS 屬性選擇器誤當成欄位類型的坑）——查 `SELECT DISTINCT FType FROM TsField WHERE FType LIKE 'InputBox%'` 確認合法值，數字欄位用 `InputBox-Integer`（整數）或 `InputBox-Double`（有小數）。
- **`TsUnit`/`TsPage`/`TsToolItem`/`TsRoleMenu` 這類 metadata 全部有 Registry/前端快取**，改完 DB 一定要 `kill-leftover.ps1` + 重啟 `server.bat`，單純重新整理網頁沒用。特別是 `TsUnit.FIcon`——連續改兩次圖示都沒重啟，畫面會一直停在最早那次寫錯的值，因為表單頁籤的圖示是讀 `TsUnit` 層級、不是 `TsPage` 層級的快取。
- **選單只掛 `TsMenu` 不夠，還要 `TsRoleMenu` 才會讓非 Administrator 帳號看得到**——Administrator 角色本身會跳過選單隱藏判斷，只用這個帳號測試「選單看不看得到」會得到偽陰性結果。
  ```sql
  INSERT INTO TsRoleMenu (FRoleId,FMenuId) VALUES
   ('00000000-0000-0000-1004-000000000002',@MENU),  -- 系統管理員
   ('17f4e3d5-be30-0905-4088-3c6aa7bb54e5',@MENU),   -- 租戶管理員
   ('00000000-0000-0000-1004-100000000001',@MENU),   -- 員工
   ('00000000-0000-0000-1004-000000000001',@MENU);   -- 預設角色
  ```
- **`TsRolePrivilege.FGlobal` 要明確設 `b'1'`，不能留 `NULL`**——`NULL` 會落到 identity-scoped 檢查路徑，對一般沒有部門歸屬欄位的業務表一律判定沒有權限，即使 `TsRolePrivilege` 那筆資料明明存在也一樣被擋（`Qs.Privilege.CannotCreate` 之類的錯誤）。
- **點擊觸發自訂 JS（`TsPage.FLoadHandler` 指到一支外部 `.js` 檔）在這個部署上實測沒有真正生效過**（連系統既有的 USB 控管模組按鈕都一樣，只是沒人點過才沒被發現）——需要按鈕觸發邏輯時，直接把可執行的 JS 字串寫進 `TsToolItem.FDefaultEventHandler`（框架的 `Jui.util.execute()` 偵測到字串裡有 `(` 就直接 `eval()`），不要依賴 `FLoadHandler`：
  ```sql
  UPDATE TsToolItem SET FDefaultEventHandler=
    "if(list.getSelectedKeys()[0]==null){Jui.message.alert($text('Public.SelectAlert'));}else{CommonBusiness.openViewPage('Ecp.Xxx',list.getSelectedKeys()[0]);}"
   WHERE FId=@TOOL_ID;
  ```
  `CommonBusiness.openViewDialog(unitCode,entityId)` 強制彈窗；`CommonBusiness.openViewPage(unitCode,entityId)` 會自動判斷開新分頁還是彈窗（在 iframe/對話框裡就用彈窗，否則開分頁）——使用者要「不是彈出視窗」時用後者。
- **`democustomerorder_unit.sql` 這類腳本開頭都設計成冪等**（先 DELETE 舊資料、`DROP TABLE IF EXISTS` 再重建），可以放心重跑，但重跑會清空該模組的真實資料（僅適合測試/示範用途，正式資料不要用這招重置）。

---

## Conformance Addendum

## When to Use
在 aipower/ECP（Quicksilver 框架）建立母子表單（一對多，類似 Access「主表單含子表單」）的兩種原生做法——(1) 頁籤式：TsRelation + 從屬 TsPage（FMasterUnitId+FIsSlavePage），點頁籤才切換，適合「客戶→多筆訂單」這種鬆散關聯；(2) 上下堆疊式：TsUnit 本身的 FIsSlaveUnit+FMasterUnitId+FMasterField（真正的「從屬單元」），母表單同一頁下方自動內嵌可編輯明細表格（新增/刪除/上升/下降/置頂/置底全部框架自動產生），適合「訂單→多筆明細」「費用→多筆費用明細」這種緊密的一對多，完全比照系統內建 Ecp.Order/OrderDetail、Ecp.Expense/ExpenseDetail 反查得出。含完整可複製的 SQL 範本、六大核心 Java 樣板、以及一系列反查/實測踩過的坑（TsUnit.FIcon 快取、母表單頁籤的 FUnitId 故意設成子單元 ID 這個違反直覺的關鍵、InputBox-Number 不存在、TsRoleMenu 選單授權、embedded MariaDB port 每次重啟會變）。當使用者要求「做一個母子表單」「一對多表單」「像 Access 那種 subform」「訂單/費用 那種上下佈局的表單」「一頁同時看到主表跟明細」時使用。

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
