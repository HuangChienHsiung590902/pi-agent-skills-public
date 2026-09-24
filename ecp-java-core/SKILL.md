---
name: ecp-java-core
description: Chainsea ECP / aipower Java 開發的六大核心分層設計準則（Model / Dao / Service / Action / Api / Home），建立在 Quicksilver 框架之上。新增或修改任何 aipower 模組程式碼前必讀，務必嚴格遵守命名、分層與 Context 規範。
triggers:
  - ecp java
  - aipower 開發
  - aipower 模組
  - 六大核心
  - quicksilver
  - EntityAction
  - EntityService
  - EntityDao
  - 寫 aipower code
  - aipower module
argument-hint: "[module name | layer: model/dao/service/action/api/home]"
---

# ECP / aipower Java 六大核心設計準則

aipower（ECP）的 Java 後端建立在 **Quicksilver 框架**（`com.jeedsoft.quicksilver`）之上。
每一個業務模組都是一個套件 `com.chainsea.ecp.<module>`，且**必須**由六個固定角色組成。
這不是建議，而是框架的硬性契約：Registry 依「命名慣例 + UNIT_ID」自動掃描、建立 proxy 並注入。
偏離慣例 = 模組無法被框架載入或注入。

> 來源：`aipower-module-base-*.jar` 反編譯實證 +
> `quicksilver-module-main-*.jar`（`base/{model,dao,service,action,api}`、`registry.Registry`、`application.module.QsModuleManager`）。

---

## 一、六大核心（每個模組固定六個角色）

以 `address` 模組為例，套件 `com.chainsea.ecp.address`：

| # | 核心 | 類別範例 | 繼承自（Quicksilver base） | Context | 回傳型別 | 職責 |
|---|------|----------|---------------------------|---------|----------|------|
| 1 | **Model 模型** | `AddressModel` | `EntityModel`→`BaseModel`→`Record` | — | — | 純資料載體；UUID 主鍵；可由 `Record`/`ResultSet`/`JSONObject` 建構 |
| 2 | **Dao 資料存取** | `AddressDao` + `AddressDaoImpl` | `EntityDao<T>` / `EntityDaoImpl<T>` | `DaoContext` | Model / DataSet | 只負責持久化（SQL / Elasticsearch），**零業務邏輯** |
| 3 | **Service 業務邏輯** | `AddressService` + `AddressServiceImpl` | `EntityService<T>` / `EntityServiceImpl<T>` | `ServiceContext` | Model / DataSet / UUID | 業務規則、交易、權限、快取邊界；透過 `Home.getDao()` 取 Dao |
| 4 | **Action 網頁動作** | `AddressAction` + `AddressActionImpl` | `EntityAction<T>` / `EntityActionImpl<T>` | `ActionContext` | `DataResult` / `PageResult` / `FileResult` | 後台網頁 MVC 端點；薄轉接層 |
| 5 | **Api 對外介面** | `AddressApi` + `AddressApiImpl` | `EntityApi<T>` / `EntityApiImpl<T>` | `ApiContext` | `JsonResult` | REST / 行動裝置 / 外部整合 JSON 端點；薄轉接層 |
| 6 | **Home 模組樞紐** | `AddressHome` | （無，純 static 定位器） | — | — | 持有 `UNIT_ID`、`CACHE_REGION` 與 static `dao`/`service`，由 Registry 注入 |

> 並非所有模組都需要 `Api`（base jar 約 45 個模組有 Action，僅 20 個有 Api）。
> **Api 只在需要對外/行動/整合 JSON 時建立**；後台功能用 Action 即可。其餘五核心（Model/Dao/Service/Action/Home）為標配。

### 套件結構（強制）
```
com/chainsea/ecp/<module>/
├── <Module>Home.java                 ← 6. Home（套件根，模組樞紐）
├── model/<Module>Model.java          ← 1. Model
├── dao/<Module>Dao.java              ← 2. Dao 介面
├── dao/impl/<Module>DaoImpl.java     ←    Dao 實作
├── service/<Module>Service.java      ← 3. Service 介面
├── service/impl/<Module>ServiceImpl.java
├── action/<Module>Action.java        ← 4. Action 介面
├── action/impl/<Module>ActionImpl.java
├── api/<Module>Api.java              ← 5. Api 介面（選用）
└── api/impl/<Module>ApiImpl.java
```

---

## 二、三大支撐支柱（讓六大核心運作的機制）

### A. Context（每層各有專屬 Context，貫穿全鏈）
`ActionContext` → `ApiContext` → `ServiceContext` → `DaoContext`，由框架 `ContextFilter` 在請求進入時建立並逐層傳遞。
**Context 內含租戶（tenant）、登入者、權限、交易、語系等。**

- **每一個方法的第一個參數都是該層 Context。** 不允許省略。
- 一切「我是誰、哪個租戶、有什麼權限」**都從 Context 取得**。
- **嚴禁**在 Service/Dao 直接碰 `HttpServletRequest` / `HttpSession` / `ThreadLocal` 取使用者資訊。

### B. Result（Action / Api 的標準回傳）
- Action：`DataResult`（JSON 資料）、`PageResult`（轉址/JSP 頁面）、`FileResult`（檔案下載）。
- Api：`JsonResult`（統一 JSON 封裝）。
- **不要**自己寫 `response.getWriter().print(...)`；一律回傳對應 Result 物件。

### C. Registry + 動態 Proxy（為何不能 `new`）
`com.jeedsoft.quicksilver.registry.Registry` 在啟動 / `initializeUnit(UNIT_ID)` 時：
1. 依 `UnitModel`（對應 DB 的 `TsUnit`，由 `UNIT_ID` 串接）找出每個模組的各層 impl 類別；
2. 將 impl 包成**動態 proxy**（套上交易、快取、權限、多租戶等 AOP）；
3. 註冊進 `daoMap`/`serviceMap`/`actionMap`/`apiMap`，並回填到各模組的 `Home.setDao()`/`setService()`。

**推論（必守）：**
- **永遠不要 `new XxxServiceImpl()` / `new XxxDaoImpl()`**——那會繞過 proxy，失去交易/快取/權限/租戶隔離。
- 取相依物件一律走 `XxxHome.getService()` / `XxxHome.getDao()`；跨模組則用 `Registry.getEntityService(otherUnitId)`。
- 因為要被 proxy 包裝，**每層都必須「介面 + impl 分離」**，不可只寫一個具體類別。

### D. 模組描述檔（封裝與相依）
每個模組 jar 內含 `QS-MODULE/module.xml`：宣告 `name`、`version`、`dependencies`、`fileOperations`；
`QS-MODULE/data/sql/default/init.xml` 放初始 schema/資料。`QsModuleManager` 依相依關係拓樸排序載入。
新模組要能被載入，**必須**有正確的 `module.xml` 與相依宣告。

---

## 三、鐵則（寫 / 改 aipower code 必守）

1. **命名即契約**：類別/套件名稱嚴格照 `<Module>` + 層級後綴（`...Model/Dao/DaoImpl/Service/.../Home`）。Registry 靠名稱解析，改名 = 注入失敗。
2. **單向分層，不可跨層、不可逆向**：
   `Action / Api → Service → Dao → Model`。
   - Action/Api **絕不**直接呼叫 Dao 或寫 SQL。
   - Service **絕不** import 任何 `action`/`api`/`web` 套件，也不碰 HTTP。
   - Dao **絕不**含業務判斷。
3. **每個方法都吃對應 Context，並從中取租戶/使用者/權限**；不得自行從全域抓。
4. **相依一律走 Home / Registry，嚴禁 `new` impl**（破壞 proxy）。
5. **職責純度**：
   - Model＝資料，零邏輯（除了欄位轉型 getter/setter）。
   - Dao＝持久化，零業務。
   - Service＝業務 + 交易邊界（交易、快取失效、權限檢核、跨模組協調）。
   - Action/Api＝薄轉接：解析參數 → 呼叫 Service → 包成 Result。**邏輯不准寫在這層。**
6. **介面 + impl 永遠分離**（讓框架可代理）。
7. `UNIT_ID`（對應 `TsUnit`）與 `CACHE_REGION`（如 `ecp_address`）**只**定義在 Home，當作模組常數來源。
8. **沿用既有 base 階層**：若同類功能已有中介基底（例：通訊錄類模組繼承 `communication` 的 `CommunicationModel/Service/Dao`），新模組應繼承該中介基底而非直接繼承 Quicksilver base，以複用共通行為。
9. **新模組**：建立六大核心 + `module.xml`（宣告相依）+ `init.xml`（schema），並在 DB 補上 `TsUnit`/`TsMenu`/`TsPage` 元資料（見 `ecp-schema` skill）。

---

## 四、最小骨架範例（新增模組 `foo`）

```java
// 6. Home —— 模組樞紐（套件根）
package com.chainsea.ecp.foo;
import com.chainsea.ecp.foo.dao.FooDao;
import com.chainsea.ecp.foo.service.FooService;
import java.util.UUID;
public class FooHome {
    public static final UUID   UNIT_ID      = UUID.fromString("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"); // = TsUnit.id
    public static final String CACHE_REGION = "ecp_foo";
    private static FooDao dao;          // 由 Registry 啟動時注入
    private static FooService service;  // 由 Registry 啟動時注入
    public static FooDao getDao()              { return dao; }
    public static void   setDao(FooDao d)      { dao = d; }
    public static FooService getService()      { return service; }
    public static void   setService(FooService s) { service = s; }
}

// 1. Model
package com.chainsea.ecp.foo.model;
import com.jeedsoft.quicksilver.base.model.EntityModel;
import com.jeedsoft.common.advanced.db.dataset.Record;
import org.json.JSONObject;
import java.sql.ResultSet;
public class FooModel extends EntityModel {
    public FooModel() {}
    public FooModel(Record r)     { super(r); }
    public FooModel(ResultSet rs) { super(rs); }
    public FooModel(JSONObject j) { super(j); }
    // 型別化欄位存取，無業務邏輯：
    public String getName()          { return getString("name"); }
    public void   setName(String v)  { put("name", v); }
}

// 2. Dao（介面 + impl）
package com.chainsea.ecp.foo.dao;
import com.jeedsoft.quicksilver.base.dao.EntityDao;
import com.chainsea.ecp.foo.model.FooModel;
public interface FooDao extends EntityDao<FooModel> {}

package com.chainsea.ecp.foo.dao.impl;
import com.jeedsoft.quicksilver.base.dao.impl.EntityDaoImpl;
import com.chainsea.ecp.foo.dao.FooDao;
import com.chainsea.ecp.foo.model.FooModel;
public class FooDaoImpl extends EntityDaoImpl<FooModel> implements FooDao {}

// 3. Service（介面 + impl）—— 透過 Home 取 Dao，吃 ServiceContext
package com.chainsea.ecp.foo.service;
import com.jeedsoft.quicksilver.base.service.EntityService;
import com.chainsea.ecp.foo.model.FooModel;
public interface FooService extends EntityService<FooModel> {}

package com.chainsea.ecp.foo.service.impl;
import com.jeedsoft.quicksilver.base.service.impl.EntityServiceImpl;
import com.chainsea.ecp.foo.service.FooService;
import com.chainsea.ecp.foo.dao.FooDao;
import com.chainsea.ecp.foo.model.FooModel;
import com.chainsea.ecp.foo.FooHome;
public class FooServiceImpl extends EntityServiceImpl<FooModel> implements FooService {}
// ⚠ 實證修正（2026-06-29, Lab aipower）：此版 EntityServiceImpl 無 getDao() 可覆寫，
//   寫 @Override getDao() 會編譯失敗(method does not override)。ServiceImpl 留空殼即可，
//   Dao 由 Registry 依 UNIT_ID 內部注入（真實 ContactServiceImpl 亦無 getDao）。

// 4. Action（介面 + impl）—— 薄轉接，回傳 Result，吃 ActionContext
package com.chainsea.ecp.foo.action;
import com.jeedsoft.quicksilver.base.action.EntityAction;
import com.chainsea.ecp.foo.model.FooModel;
public interface FooAction extends EntityAction<FooModel> {}

package com.chainsea.ecp.foo.action.impl;
import com.jeedsoft.quicksilver.base.action.impl.EntityActionImpl;
import com.chainsea.ecp.foo.action.FooAction;
import com.chainsea.ecp.foo.model.FooModel;
public class FooActionImpl extends EntityActionImpl<FooModel> implements FooAction {}

// 5. Api（介面 + impl）—— 選用，回傳 JsonResult，吃 ApiContext
package com.chainsea.ecp.foo.api;
import com.jeedsoft.quicksilver.base.api.EntityApi;
import com.chainsea.ecp.foo.model.FooModel;
public interface FooApi extends EntityApi<FooModel> {}

package com.chainsea.ecp.foo.api.impl;
import com.jeedsoft.quicksilver.base.api.impl.EntityApiImpl;
import com.chainsea.ecp.foo.api.FooApi;
import com.chainsea.ecp.foo.model.FooModel;
public class FooApiImpl extends EntityApiImpl<FooModel> implements FooApi {}
```

繼承 `EntityXxx` 即免費獲得標準 CRUD（`getItem/getItems/create/update/delete/save/...`）。
**只在需要客製行為時覆寫**，覆寫時仍維持「Action→Service→Dao」單向呼叫與 Context 傳遞。

---

## 五、固定開發路徑（src / 編譯 / 部署 — 一律照此，不得另創它處）

所有 aipower / chainsea 客製 Java 開發**只用以下路徑**。不要在 Desktop、Documents、暫存夾或別的地方東放一個西放一個。

| 用途 | 固定路徑 |
|------|----------|
| **原始碼 src** | `C:\com\chainsea\tool\src\` |
| **編譯輸出 classes** | `C:\com\chainsea\tool\classes\` |
| **編譯 classpath** | `C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\*` |
| **aipower 部署目標** | `C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\classes\`（覆蓋 jar 內同名類別） |
| **JDK** | `C:\com\chainsea\jdk\bin\javac.exe` / `java.exe` |
| **反編譯參考（唯讀）** | `C:\com\chainsea\tool\decompile\` |

### 規範
1. **`.java` 一律放 `tool\src\`**（扁平擺放即可），檔名 = 公開類別名；檔內務必寫正確 `package com.chainsea.ecp.<module>...;`，javac 會依 package 宣告在 `tool\classes` 下自動建立目錄結構。
2. **永遠編譯到 `tool\classes`**，classpath 永遠指向 aipower 的 `WEB-INF\lib\*`（取得 Quicksilver + 模組 base）。
3. **部署 aipower 模組**：把編好的 `com\chainsea\ecp\...\*.class` 複製到 `WEB-INF\classes\`。Servlet 規範中 `WEB-INF\classes` 優先於 `WEB-INF\lib\*.jar`，故此即「覆蓋 jar 內原類別」的標準作法；改完重啟 Tomcat 生效（見 `ecp-server-startup`）。
4. **獨立工具（如 cbm-lite）**：同樣 src→classes，直接從 `tool\classes` 以 `java` 執行，不進 webapp（見 `cbm-lite` skill）。
5. 既有實例可直接參照：`tool\src\CbmLiteProcessKnowledgeActionImpl.java`、`tool\src\MikoPbxCallLogActionImpl.java`（皆為六大核心的 ActionImpl 覆寫）。

### 固定編譯指令（PowerShell）
```powershell
& "C:\com\chainsea\jdk\bin\javac.exe" -encoding UTF-8 `
  -cp "C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\*" `
  -d "C:\com\chainsea\tool\classes" `
  "C:\com\chainsea\tool\src\<YourClass>.java"

# 部署 aipower 客製類別（覆蓋 jar）：
Copy-Item -Recurse -Force "C:\com\chainsea\tool\classes\com" `
  "C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\classes\"
# 然後重啟 Tomcat
```

---

## 五之二、API 2.0 對外 REST（單元配置，2026-07-03 端到端實證）

> 官方手冊：API **1.0**（手動建 `TsLocalApi`/API組）**已過時**；改用 **2.0＝單元配置**，API 作為一層和 Action 並列，實體單元零額外註冊即有 REST。

**啟用（六大核心 + 純 DB 設定）：**
1. 寫 Api 兩層：`XxxApi extends EntityApi<Model>`、`XxxApiImpl extends EntityApiImpl<Model>`（建構子 `super(XxxHome.UNIT_ID, XxxModel.class)`），編譯部署到 `WEB-INF/classes`。
2. `TsUnit` 設 `FApiClassName='...api.impl.XxxApiImpl'` + `FApiEnabled=b'1'`。重啟 → log 出現 `Create proxy: Ecp.Xxx.Api`。
3. 實體單元**預設**支援 list/item/create/update/delete（`EntityApiImpl` 已含，子類空殼即繼承其 `@Api` 注解）。自訂方法要自己加 `@Api` 注解（不加＝不對外；子類覆寫預設 `isInherited=true` 繼承父注解）。

**URL 格式**：`/openapi/<module>/<business>/<operation>`，module/business＝**小寫 unit 編碼**（`Ecp.LunchOrder`→`ecp/lunchorder`；大寫會 404）。ApiServlet 掛在 `/openapi/*`。

**呼叫需 token**（APP/外部系統無 session）：
```
# 1. 申請 token（帳號需有「職務」身份）
POST /aipower/openapi/qs/user/token/apply
{"loginName":"HCH","password":"...","language":"zh-tw"}   -> {"tokenId":"...","_header_":{"success":true}}
# 2. 帶 token 呼叫（每次 body 放 _header_.tokenId）
POST /aipower/openapi/ecp/lunchorder/list
{"_header_":{"tokenId":"<上面拿到的>"}}                    -> {"_header_":{"success":true},"items":[...]}
```

**錯誤碼速查**：
- `404` → URL 用了大寫 unit 編碼、module/business 拆錯、或 `FApiEnabled` 沒開。
- `Qs.Token.Required` → 沒帶 tokenId（**代表路由正確、API 已上線**，不是壞掉）。
- `Qs.Account.NoIdentityType`（apply token 時）→ 該帳號缺「職務」身份；補一筆 `TsAccountIdentity`（職務類型 `564cf69e-76d6-4baf-b584-6e04c2911dae`，FEntityId 指一個職務 TsUser，如系統管理員職務 `00000000-0000-0000-1002-000000000001`）。
- `Qs.Auth.LoginFailed` → 帳密錯（用 `ecp-pwd` skill 設已知密碼）。

> 1.0 舊路線（僅維護既有時參考）：獨立類別 `com.qsapi.XxxApi.method` + DbConfig + 手建 `TsLocalApiGroup`(API組) + `TsLocalApi`(FPath/FTargetType=ApiMethod/FTarget=`Package.Class.Method`/FHandleType=Partial)。新開發一律用 2.0。

## 六、Review 檢查清單（改完 aipower code 自我驗收）
- [ ] 套件、類別命名完全符合六大核心慣例？
- [ ] 每層介面 + impl 分離？impl 都繼承對應 `EntityXxxImpl`？
- [ ] 沒有任何 `new XxxServiceImpl/DaoImpl`？相依都走 `Home`/`Registry`？
- [ ] Action/Api 沒有 SQL、沒有業務邏輯，只解析→呼叫 Service→回 Result？
- [ ] Service 沒有 import action/api/web、沒碰 HttpServletRequest？
- [ ] 每個方法都帶對應 Context，租戶/使用者/權限皆取自 Context？
- [ ] Model 純資料、無業務？
- [ ] 新模組有 `module.xml`（相依宣告）、`init.xml`、`UNIT_ID`/`CACHE_REGION` 常數，及 DB `TsUnit/TsMenu/TsPage` 元資料？
- [ ] 原始碼放 `tool\src`、編到 `tool\classes`、部署到 `WEB-INF\classes`？沒有散落在其它目錄？

## 相關 skill
- `ecp-schema`：Unit/Menu/Page 元資料、`UNIT_ID` 對應 `TsUnit`。
- `ecp-server-startup`：啟動框架、排除 MariaDB4j。
- `ecp-menu`：把新模組掛上左側選單。
- `chainsea-feature-migration`：跨部署搬移單一功能模組。

---

## Conformance Addendum

## When to Use
Chainsea ECP / aipower Java 開發的六大核心分層設計準則（Model / Dao / Service / Action / Api / Home），建立在 Quicksilver 框架之上。新增或修改任何 aipower 模組程式碼前必讀，務必嚴格遵守命名、分層與 Context 規範。

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
