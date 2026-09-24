---
name: ecp-deepseek
description: 在 aipower（現行 live 實例見 ecp-lab2-instance skill，port 22821）自建的 Ecp.DeepSeek 單元——讓 ECP 前後端呼叫 DeepSeek（chat/completions、models），並附一個自建的輕量客服台（讀寫 TcChatRoom/TcChatMessage、AI 建議回覆）。用在：呼叫/測試 DeepSeek、改 API key/model、重建或重新部署此模組、把類似的外部 LLM/REST 服務接進 ECP、或要做一個不碰複雜原生 Vue/Aiff 客服台的簡單聊天測試介面。含「為何不能用內建遠端 API」與系統參數頁顯示的 FRange 坑。
---

# ECP DeepSeek 整合（Ecp.DeepSeek 單元）

在 live 的 aipower 自建 `Ecp.DeepSeek` 單元，用原生 `HttpURLConnection` 呼叫 DeepSeek（OpenAI 相容）。功能已端到端實測通過。

> ⚠️ **這支 skill 裡的路徑最初寫在 `C:\Lab\chainsea`，該實例後來損毀，目前的 live 實例是 `C:\Lab2\chainsea`（同 port 22821）**——見 `ecp-lab2-instance` skill 了解始末。下面路徑若對不上，先確認現在的 live 實例是哪一個（通常看 port 22821 由誰佔用）。
>
> 相關：`ecp-java-core`（六核心）、`ecp-server-startup`（重啟嵌入式 mariadb 的 patch 流程）、`ecp-schema`（Ts* 元資料）、`ecp-lab2-instance`（live 實例現況）、`cbm-lite`（另一套已整合 LINE 的客服台，也可指向 DeepSeek）。

## 一、怎麼呼叫（前端 / 後端）

`.data` 端點需登入 session。回傳即 DeepSeek 原始 JSON。

```javascript
// 對話補全：args 直接當 chat/completions 的 body（至少要有 messages；model 可省，省略用系統參數）
Utility.invoke('Ecp.DeepSeek.chat', {messages:[{role:'user',content:'你好'}]}, false, function(r){
    console.log(r.choices[0].message.content);
});
// 模型清單
Utility.invoke('Ecp.DeepSeek.models', {});
```

後端 Java：`DeepSeekHome.getService().chat(ctx.getServiceContext(), argsJson)` / `.listModels(ctx.getServiceContext())`。

## 二、設定（系統參數 TsSystemParameter，即時讀，改完下一次呼叫立即生效、免重啟）

| FKey | 預設 |
|---|---|
| `deepseek.apiKey` | （你的 sk-...）|
| `deepseek.baseUrl` | `https://api.deepseek.com` |
| `deepseek.model` | `deepseek-chat`（別名→實際 deepseek-v4-flash；另有 deepseek-v4-pro）|
| `deepseek.http.timeoutMs` | `60000` |

改值最快：**系統管理→資料庫→執行 SQL**：
```sql
UPDATE TsSystemParameter SET FValue='新值' WHERE FKey='deepseek.apiKey';
```
`DeepSeekHome.getConfig` 每次呼叫都 `SELECT ... FROM TsSystemParameter WHERE FKey=?`，故即時生效。

## 三、為何不能用內建「遠端 API 組/遠端 API」（資料整合）——踩過的死路

兩個框架先天限制同時卡死 DeepSeek：
1. **送不出 `Authorization: Bearer`**：`RemoteApiUtil.invoke` 只自動設 Content-Type；body 裡的 `_header_` 只是「平台標準請求」信封、**不是 HTTP header**；群組「安全驗證方式」只有 Chainsea Token 或「其它=無」。
2. **解不了回應**：舊 `HttpQuery.getResponseJson()`（quicksilver-lib-basic-7.2.2.beta17）遇到「回應 Content-Type 無 charset（DeepSeek 回裸 `application/json`）＋請求無 Accept-Charset」就丟 `Unknown response content type`（HttpQuery.send:314），而 invoke 從不呼叫 setAcceptCharset。

→ **解法＝原生 `java.net.HttpURLConnection`**（非 HttpQuery）：自帶 Bearer、回應一律 UTF-8 讀，兩問題全繞過。此為本專案既有 pattern（cbm-lite 的 ProcessKnowledge action 也這樣）。

## 四、程式（六核心，`com.chainsea.ecp.deepseek`）

src 在 `C:\Lab\chainsea\tool\src\DeepSeek*.java`（8 檔）：
- `DeepSeekHome`：UNIT_ID=`deef5eec-0000-4001-b001-000000000001`、CACHE_REGION、`Registry.getDao/getService`、`getConfig(key,def)`（讀 TsSystemParameter）。
- `DeepSeekModel`（extends EntityModel，空殼）、`DeepSeekDao(+Impl)`（EntityDao，空殼，無業務實體但註冊需要）。
- `DeepSeekServiceImpl`：真正邏輯——`chat()`/`listModels()` 用 HttpURLConnection 打 DeepSeek。
- `DeepSeekActionImpl`：薄轉接，`chat(ActionContext)`/`models(ActionContext)` → `getService().xxx(ctx.getServiceContext(),...)` → `DataResult.putAll(resp)`。

### 重建/重新部署（改了 .java 後）
```powershell
# 1) 編譯
& "C:\Lab\chainsea\jdk\bin\javac.exe" -encoding UTF-8 `
  -cp "C:\Lab\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\*" `
  -d "C:\Lab\chainsea\tool\classes" `
  C:\Lab\chainsea\tool\src\DeepSeek*.java
# 2) 部署（覆蓋 jar 內同名類別）
Copy-Item -Recurse -Force "C:\Lab\chainsea\tool\classes\com\chainsea\ecp\deepseek" `
  "C:\Lab\chainsea\apache-tomcat\webapps\aipower\WEB-INF\classes\com\chainsea\ecp\"
# 3) 重啟 Tomcat（走 ecp-server-startup patch 流程，見下）
```

## 五、DB 元資料（idempotent；DB=`default`；嵌入式 mariadb port 每次啟動會變，用下法抓）
```bash
PORT=$(powershell -NoProfile -Command "Get-CimInstance Win32_Process -Filter \"name='mariadbd.exe'\" | Select -Expand CommandLine" | tr ' ' '\n' | grep -oE '^--port=[0-9]+' | grep -oE '[0-9]+')
MYSQL="C:\Lab\chainsea\apache-tomcat\temp\MariaDB4j\base\bin\mysql.exe"   # root 無密碼
```
```sql
-- 單元（新單元/改 class 要重啟 Tomcat 才生效；Registry log 出現 "Register unit: Ecp.DeepSeek"）
CREATE TABLE IF NOT EXISTS TcDeepSeek (FId varchar(36) NOT NULL, PRIMARY KEY(FId));
INSERT INTO TsUnit (FId,FCode,FName,FModuleId,FTable,FOpenMode,FHomeClassName,FDaoClassName,FServiceClassName,FActionClassName)
VALUES ('deef5eec-0000-4001-b001-000000000001','Ecp.DeepSeek','DeepSeek','00000000-0000-0000-0008-020000000010','TcDeepSeek','System',
 'com.chainsea.ecp.deepseek.DeepSeekHome','com.chainsea.ecp.deepseek.dao.impl.DeepSeekDaoImpl',
 'com.chainsea.ecp.deepseek.service.impl.DeepSeekServiceImpl','com.chainsea.ecp.deepseek.action.impl.DeepSeekActionImpl');
INSERT INTO TsField (FId,FUnitId,FName,FTitle,FType,FVisible,FRequired,FReadOnly,FQueryable)
VALUES ('deef5eec-0000-4001-b001-0000000000f0','deef5eec-0000-4001-b001-000000000001','FId','ID','InputBox-Key',b'0',b'0',b'1',b'0');

-- 參數值
INSERT INTO TsSystemParameter (FId,FKey,FValue) VALUES
 (UUID(),'deepseek.apiKey','sk-...'),(UUID(),'deepseek.baseUrl','https://api.deepseek.com'),
 (UUID(),'deepseek.model','deepseek-chat'),(UUID(),'deepseek.http.timeoutMs','60000');
```

### 讓參數出現在「系統參數」頁（重要坑）
只寫 `TsSystemParameter`（值）不會顯示——要建 `TsParameterGroup`(level-1 root + level-2 子群) + `TsParameterDefinition`，**且 `FRange` 必須含 `System`**。系統參數頁的 `getEditJson`→`ParameterDefinitionDao.getVisibleItems` 用 `FVisible=1` 撈、再用 `locate('System',FRange)>0` 依 scope 過濾；`FRange` 為 NULL＝不顯示（Duty/Company/Tenant 同理）。
```sql
INSERT INTO TsParameterGroup (FId,FName,FParentId,FTreeLevel,FTreeSerial,FIndex) VALUES
 ('deef5eec-0000-4001-b001-0000000000a0','DeepSeek',NULL,1,'005',5),
 ('deef5eec-0000-4001-b001-0000000000a1','DeepSeek 設定','deef5eec-0000-4001-b001-0000000000a0',2,'005.001',1);
INSERT INTO TsParameterDefinition (FId,FName,FParameterGroupId,FCode,FType,FRange,FRequired,FIndex,FVisible,FRowSpan,FColSpan) VALUES
 (UUID(),'DeepSeek API Key','deef5eec-0000-4001-b001-0000000000a1','deepseek.apiKey','InputBox-Text','System',b'0',1,b'1',1,1),
 (UUID(),'DeepSeek Base URL','deef5eec-0000-4001-b001-0000000000a1','deepseek.baseUrl','InputBox-Text','System',b'0',2,b'1',1,1),
 (UUID(),'DeepSeek 模型','deef5eec-0000-4001-b001-0000000000a1','deepseek.model','InputBox-Text','System',b'0',3,b'1',1,1),
 (UUID(),'HTTP 逾時(ms)','deef5eec-0000-4001-b001-0000000000a1','deepseek.http.timeoutMs','InputBox-Text','System',b'0',4,b'1',1,1);
```
- 定義的 `FCode` 必須 = `TsSystemParameter.FKey`。
- 定義是**即時 DB 讀**，改完重新整理頁面就顯示（不需重啟）。
- **值欄位快取坑**：系統參數頁「現值」是啟動時載入的快取，只載 FRange 含 System 的定義。若先建定義(FRange=NULL)→啟動→再補 FRange=System，欄位會顯示**空白**（值快取沒帶到）。修法：重啟一次，或直接在表單貼值 + 保存。**⚠ 別把空白表單直接保存**——會把 TsSystemParameter 值清空、弄壞 chat。

## 六、重啟（嵌入式 MariaDB4j，走 ecp-server-startup patch）
jar 已 patch（`patch_install.py` 對 `C:\Lab\chainsea` 下所有 `mariaDB4j-core-*.jar` idempotent）。停 Lab 的 Tomcat java（port 22821 owner）+ mariadbd（cmdline 含 `Lab\chainsea`），再 `Start-Process C:\Lab\chainsea\server.bat -WorkingDirectory C:\Lab\chainsea`，等約 15–40s，log 看 `Register unit: Ecp.DeepSeek` → `Quicksilver startup in ... ms`。

## 七、驗證（curl 自助登入，繞過 Chrome）
`.page` HTML 是前端 ajax 拉資料的空骨架，**curl grep 頁面是假陰性**；驗證用「跑真實資料 action/SQL」或看真 UI。
登入拿 session：`Qs.Misc.getLoginPublicKey`→RSA(PKCS1v15) 加密密碼→POST `Qs.OnlineUser.login`（args:{loginName,password:enc,language:'zh-tw',extraArgs:{}}）→再打 `Ecp.DeepSeek.chat.data`。Python 範例（需 `cryptography`）存於本 skill 開發時的 scratchpad `ds_test.py`。
- 路由存在但未登入 → `Qs.Auth.SessionInvalid`（非 404，代表單元已註冊）。
- 登入成功 → login 回 `{}`（空=成功）。

## 八、輕量客服台（自建，繞開原生複雜的 Vue/Aiff ServiceConsult）

原生「服務諮詢/文字客服」（`Aipower.ServiceConsult.Main`）是 Vue.js + 內嵌「Aiff」子平台，需要完整 workgroup/queue/agent 狀態機才能正確渲染對話，塞測試資料進去不保證顯示正常（見 `ecp-lab2-instance` skill）。若只是要「有個能測 DeepSeek AI 建議回覆的聊天畫面」，改用這支自建的輕量客服台，直接讀寫 ECP 原生的 `TcChatRoom`/`TcChatMessage` 表（跟 `cbm-lite` 讀的是同一組表），不受 Aiff 那層限制。

### 後端：擴充 DeepSeekService/Action（`com.chainsea.ecp.deepseek`）

`DeepSeekServiceImpl` 新增（皆用 `Executor.getInstance("default")` 直接 SQL，符合 Service 層可以直接操作 DB 的慣例）：
- `getChatRooms(ctx)` — 依最後訊息時間排序的房間清單（`SELECT r.FId,r.FName,(子查詢取最後一則內容/時間) FROM TcChatRoom r`）。
- `getChatMessages(ctx, roomId)` — 該房間全部訊息，依時間排序。
- `sendChatMessage(ctx, roomId, content)` — `INSERT INTO TcChatMessage`（`FSenderId=ctx.getUserId()`、`FCategory='Inner'`＝客服端）。
- `suggestReply(ctx, roomId)` — 讀歷史訊息組成 `messages[]`（`FCategory='Outer'`→`role:user`，其餘→`role:assistant`），加一句系統 prompt，呼叫既有的 `chat()`，回傳 `choices[0].message.content`。

`DeepSeekActionImpl` 對應薄轉接：`getChatRooms`/`getChatMessages`/`sendChatMessage`/`suggestReply`（都回 `DataResult`），另加 `prepareConsole(ctx)` 回空 `PageResult` 給頁面用。

`TcChatMessage.FCategory`：`Outer`＝客戶來訊，其餘（`Inner`等）＝客服/系統訊息——這是 `cbm-lite` 也在用的既有慣例，兩邊資料相容、可以互看。

### 前端：自訂 JSP/JS/CSS（非 Jui List/Form 樣板，比照原生 Chat.jsp 走自訂 div 佈局）

檔案：`<webapp>/ecp/page/deepseek/DeepSeekConsole.{jsp,js,css}`（房間清單＋訊息串＋回覆框＋「AI 建議」鈕，雙欄簡單佈局）。

**⚠️ JSP 內嵌中文字一定要宣告 UTF-8**，否則 Jasper 用預設編碼（非 UTF-8）編譯 JSP 原始碼、把字面中文字元弄成亂碼（`AI å»ºè°`這種）。JSP **第一行**：
```jsp
<%@page contentType="text/html;charset=UTF-8" pageEncoding="UTF-8"%>
<%@include file="../../../quicksilver/page/include/Initialize.jsp"%>
```
JSP 檔案改動**不需重啟 Tomcat**（Jasper 下次請求自動重新編譯），瀏覽器重新整理即可看到效果；但 Action/Service 的 `.java` 改動仍需重新編譯部署＋重啟 Tomcat。

JS 用標準 ECP 慣例：`var DeepSeekConsole = {...}` 物件字面量，`Utility.invoke('Ecp.DeepSeek.xxx', args, false, callback)` 呼叫後端，不自造 `fetch`。

**已知簡化（未做）**：沒有輪詢機制——只有點擊房間/送出/重整時才抓資料，不會像 `cbm-lite` 的 `/agent` 主控台那樣每幾秒自動偵測新訊息。若要能即時看到外部（如 LINE）進來的新訊息，要嘛手動重整，要嘛之後補上 `setInterval` 輪詢 `getChatMessages`。

### 頁面／選單註冊範本

```sql
INSERT INTO TsPage (FId,FCode,FName,FUrl,FType,FUnitId,FVisible,...)
VALUES ('<uuid>','Ecp.DeepSeek.Console','DeepSeek 客服台','ecp/page/deepseek/DeepSeekConsole.jsp','Other','<DeepSeek UNIT_ID>',1,...);
-- 選單掛在子目錄下（照 ecp-menu skill），例：辦公自動化→訊息管理→DeepSeek 客服台
```

---

## Conformance Addendum

## When to Use
在 aipower（現行 live 實例見 ecp-lab2-instance skill，port 22821）自建的 Ecp.DeepSeek 單元——讓 ECP 前後端呼叫 DeepSeek（chat/completions、models），並附一個自建的輕量客服台（讀寫 TcChatRoom/TcChatMessage、AI 建議回覆）。用在：呼叫/測試 DeepSeek、改 API key/model、重建或重新部署此模組、把類似的外部 LLM/REST 服務接進 ECP、或要做一個不碰複雜原生 Vue/Aiff 客服台的簡單聊天測試介面。含「為何不能用內建遠端 API」與系統參數頁顯示的 FRange 坑。

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
