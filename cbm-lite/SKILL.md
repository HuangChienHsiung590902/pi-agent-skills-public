---
name: cbm-lite
description: cbm-lite LINE 客服系統總覽——standalone server（CbmLiteServer.java，port 12621）的編譯部署、設定、DB 注冊陷阱、LINE webhook 整合、AI↔真人客服模式切換邏輯、文字客服台、LIFF。合併自原本分散的 cbm-line／cbm-line-agent／cbm-agent-console 四個 skill。
triggers:
  - cbm-lite
  - cbm lite
  - knowledge server
  - 知識庫伺服器
  - cbm設定
  - line webhook
  - line gateway
  - LINE機器人
  - line bot
  - line flex
  - 真人客服
  - 轉人工
  - 文字客服
  - agent mode
  - escalate
  - 客服台
  - agent console
  - webchat
  - cbm-proxy
argument-hint: "[compile|restart|status|settings-ui|db-register|test|tune|deploy]"
---

# cbm-lite Skill — LINE 文字客服系統總覽

> 本 skill 合併了原本的 cbm-line（webhook 基礎）、cbm-line-agent（AI↔真人切換邏輯）、cbm-agent-console（客服台/proxy 架構）。姊妹 skill：`cbm-richmenu`（rich menu 建立與換 channel 搬遷，原名 cbm-richmenu-build + cbm-richmenu-channel-migrate）、`ecp-knowledge`（知識庫 CRUD）、`cbm-agent-ai-suggest`（AI 建議回覆功能）。
>
> ⚠️ **檔案路徑會變**：這套系統的實際部署路徑（`C:\com\chainsea` vs `C:\Lab2\chainsea`、CbmLiteServer.java 原始碼確切位置等）會隨遷移調整，不要盡信本文件裡任何寫死的路徑，動手前先用 `es`/`rg` 現場確認一次。目前已知**至少存在過** `C:\Lab2\cbm-lite\src\` 與 `C:\Lab2\chainsea\tool\src\` 兩種原始碼擺放方式的記錄，哪個是當下正確答案要現場查。本文其餘章節的邏輯/架構知識不受路徑漂移影響，仍然有效。

## 目錄結構（曾經的擺放方式，供搜尋參考）

```
<cbm-lite 根目錄>\
├── src\CbmLiteServer.java              ← 獨立 server 原始碼
├── classes\com\chainsea\ecp\cbmlite\   ← 編譯輸出
├── config\cbm-lite.properties          ← 所有設定（port/LINE/LLM/LIFF…）
├── config\cbm-lite-replies.conf        ← LINE 關鍵字自動回覆規則
├── logs\cbm-lite.log                   ← 執行 log（server.bat 導向；已知這個 FileHandler log 可能很久沒 flush，排查即時問題請用手動 `-RedirectStandardError` 導出的 log，不要只看這個檔案）
└── logs\aiproxy.log
```

**ECP Action（Tomcat 用，讓 aipower 選單/知識庫功能能呼叫到 cbm-lite）：**
```
<chainsea 根目錄>\tool\src\CbmLiteProcessKnowledgeActionImpl.java
  → 編譯到 tool\classes\
  → 部署到 WEB-INF\classes\com\chainsea\ecp\processknowledge\action\impl\
```

**設定 UI（Tomcat 用）：**
```
<chainsea 根目錄>\apache-tomcat\webapps\aipower\ecp\page\cbmlite\
├── CbmLiteSettings.jsp   ← 需要 <%@ page pageEncoding="UTF-8" %> 第一行
└── CbmLiteSettings.js
```

## 啟動

cbm-lite 是獨立 Java process（監聽 port 12621），跟 chainsea/Tomcat 分開啟動，順序通常是：MariaDB → chainsea(Tomcat) → cbm-lite（等 Tomcat 通了才啟動，因為 cbm-lite 的 classpath 會用到 Tomcat webapp 底下的 WEB-INF/lib）。若有 `start-all.bat` 之類的統一入口腳本，優先用它而不是分開手動啟動三個服務。

cbm-lite classpath 概念：`<cbm-lite classes>;<chainsea Tomcat aipower WEB-INF/classes>;WEB-INF/lib/*`

## 編譯流程

### A. CbmLiteServer（獨立 server 本體）

```powershell
& "<chainsea>\jdk\bin\javac.exe" -encoding UTF-8 `
  -d "<cbm-lite classes 目錄>" `
  -cp "<chainsea>\apache-tomcat\webapps\aipower\WEB-INF\lib\*" `
  "<CbmLiteServer.java 實際路徑，現場確認>"

# 也需部署到 WEB-INF（ECP action 用到 CbmLiteServer 內的 Home 類別）
Copy-Item -Recurse -Force `
  "<cbm-lite classes>\com\chainsea\ecp\cbmlite\" `
  "<chainsea>\apache-tomcat\webapps\aipower\WEB-INF\classes\com\chainsea\ecp\cbmlite\"
```

編譯後重啟 cbm-lite process（見下方「重啟坑」，不要只 `Stop-Process` 就以為結束了）。

**另一種等效啟動方式**（不透過 start-cbm-lite.bat，直接用 java 指令跑）：
```powershell
$cp = "tool\classes;apache-tomcat\webapps\aipower\WEB-INF\classes;apache-tomcat\webapps\aipower\WEB-INF\lib\*"
Start-Process "<chainsea>\jdk\bin\java.exe" -ArgumentList "-cp `"$cp`" com.chainsea.ecp.cbmlite.CbmLiteServer" `
  -WorkingDirectory "<chainsea 根目錄>" -WindowStyle Minimized `
  -RedirectStandardOutput "<chainsea>\cbm-lite.log" -RedirectStandardError "<chainsea>\cbm-lite.err.log"
```
DB 連線用 `localhost:3306`（**不是** `127.0.0.1`，某些 embedded MariaDB 設定下用 127.0.0.1 會 error 138）。

### B. CbmLiteProcessKnowledgeActionImpl（ECP Action）

```powershell
& "<chainsea>\jdk\bin\javac.exe" -encoding UTF-8 `
  -d "<chainsea>\tool\classes" `
  -cp "<chainsea>\apache-tomcat\webapps\aipower\WEB-INF\lib\*;<chainsea>\apache-tomcat\webapps\aipower\WEB-INF\classes" `
  "<chainsea>\tool\src\CbmLiteProcessKnowledgeActionImpl.java"

Copy-Item -Force `
  "<chainsea>\tool\classes\com\chainsea\ecp\processknowledge\action\impl\CbmLiteProcessKnowledgeActionImpl.class" `
  "<chainsea>\apache-tomcat\webapps\aipower\WEB-INF\classes\com\chainsea\ecp\processknowledge\action\impl\"
```

編譯 B 後**必須重啟 Tomcat**（跟編譯 A 不同，A 只需重啟 cbm-lite process）。

## Config Hot Reload（不重啟 process）

```powershell
Invoke-RestMethod "http://127.0.0.1:12621/admin/reload" -Method POST
```

ECP 設定 UI 的 `cbmLiteSaveSettings` 批次存檔後會自動呼叫一次。但改 `cbm.lite.liff.*`、程式邏輯本身、或影響簽章驗證的 `channelSecret` 這類值，**hot reload 不夠，需要真正重啟 process**（Java 啟動時才讀入某些欄位）。

## ECP 設定 UI

路徑：**系統管理 → 進階設定 → CBM-Lite 設定**

| Action 方法 | 用途 |
|-------------|------|
| `cbmLitePrepareSettings` | 載入頁面（PageResult） |
| `cbmLiteGetSettings` | 讀取所有 key（35 項） |
| `cbmLiteSaveSettings` | **批次儲存**（單一 call，同步寫檔，呼叫一次 reload） |
| `cbmLiteSaveSetting` | 單 key 儲存（保留相容，有 synchronized） |
| `cbmLiteTestObsidian` | 測試 Obsidian Local REST API 連線，回傳 ms + 檔案數 |

群組：伺服器 / 知識庫 / Obsidian / MariaDB 知識庫 / LLM / LINE / Agent 真人客服 / LIFF

敏感欄位眼睛遮罩：`obsidian.apiKey`、`llm.apiKey`、`line.channelSecret`、`line.channelAccessToken`

長文字 textarea：`systemPrompt`、`escalatePrompt`、`welcomeMessage`、`agent.*Message`

## ⚠️ 設定 UI 的四個陷阱

### 陷阱 1：並發存檔 → race condition → properties 檔案損毀

`doSave` 若同時送出多個 request，`saveConfigKey` 會並發讀寫同一個 `.properties`，
後寫的執行緒用舊資料覆蓋先寫的結果 → 大部分 key 消失，多行 value 炸開。

→ **已修正**：`doSave` 改成 `cbmLiteSaveSettings`（batch），單一 HTTP call；
  Java 側用 `synchronized (CbmLiteProcessKnowledgeActionImpl.class)` 保護寫檔。

### 陷阱 2：textarea 換行字元寫入 .properties → 格式炸裂

`agent.startMessage` 等多行欄位在瀏覽器送出的是實際 `\n`（換行符），
直接寫進 `.properties` 會把一個 key 拆成多行 → 整個檔案損毀。

→ **已修正**：`saveConfigKey` 寫入前先做跳脫：
```java
String escaped = value
    .replace("\\", "\\\\")
    .replace("\r\n", "\\n")
    .replace("\r",   "\\n")
    .replace("\n",   "\\n");
```

### 陷阱 3：JSP 直接寫中文字需加 pageEncoding 宣告

Tomcat 預設用 ISO-8859-1 讀 JSP 原始碼，直接寫中文會亂碼（`è¨­å®`）。
→ JSP **第一行**必須是：`<%@ page pageEncoding="UTF-8" %>`（在 `<%@include` 之前）

### 陷阱 4：properties 檔案損毀的復原方式

若 `cbm-lite.properties` 已損毀（被 race condition 破壞），
從本文「Config 參考」章節手動還原各 key，重點是 multiline value 用 `\n` 表示換行：
```properties
cbm.lite.agent.startMessage=第一行\n第二行
```

## DB 注冊 TsPage / TsMenu

環境重建時需重新插入（設定頁在 aipower 選單裡的入口）。**注意三個陷阱：**

### ⚠️ UUID 只能含 hex 字元

手寫 UUID 如 `cbm00000-...` 的 `m` 不合法，Quicksilver 解析時拋 `Error at index 2 in: 'cbm00000'`。
→ 一律用 `UUID.randomUUID().toString()` 產生。

### ⚠️ 直接 JDBC INSERT 後 FTreeLevel/FTreeSerial 會是 NULL

Quicksilver ORM 計算這兩欄；直接 INSERT 跳過計算 → 選單樹不顯示。
→ INSERT 後必須手動補：

```sql
UPDATE TsMenu SET
  FTreeLevel  = 3,
  FTreeSerial = '004.002.013',
  FTabTitleSource = NULL,
  FSubMenuSource  = NULL,
  FHideInMainMenu = NULL
WHERE FName = 'CBM-Lite 設定';
```

### ⚠️ Tomcat 啟動時 marialocal driver 不可用

用 JDBC 工具類操作 DB 時，若 Tomcat 已啟動（embedded MariaDB 被 Tomcat 鎖住），
不可用 `marialocal` driver（會報 `ibdata1 must be writable`）。
→ 改用：`jdbc:mariadb://localhost:3306/default`，driver `org.mariadb.jdbc.Driver`，密碼空。

### 已知的 UUID 與位置（單一部署上曾實測的值，遷移後需重新確認）

| 名稱 | UUID |
|------|------|
| TsPage（CBM-Lite 設定） | `8afc85f6-e9d7-4e2f-9aaf-e0f1425ff820` |
| TsMenu（CBM-Lite 設定） | `896a79f4-0079-49d9-9072-323203d201a1` |
| 進階設定（parent menu） | `00000000-0000-0000-0008-990000000020` |
| Ecp.ProcessKnowledge UNIT_ID | `fd65ef8a-17e8-4096-afb0-8c2d459b1e5c` |
| AI Proxy 設定 TsMenu | `a1b2c3d4-e5f6-7890-abcd-ef1234567803`（FIndex=12, serial=004.002.012） |

CBM-Lite 設定：FIndex=13，FTreeLevel=3，FTreeSerial=`004.002.013`

DB 寫入後**不需重啟 Tomcat**，重整頁面選單即出現。

### 重建 SQL 範本

```java
Class.forName("org.mariadb.jdbc.Driver");
try (Connection cn = DriverManager.getConnection(
        "jdbc:mariadb://localhost:3306/default", "root", "")) {
    String pageId = UUID.randomUUID().toString();
    String menuId = UUID.randomUUID().toString();

    cn.createStatement().executeUpdate(
        "DELETE FROM TsMenu WHERE FName='CBM-Lite 設定'");
    cn.createStatement().executeUpdate(
        "DELETE FROM TsPage WHERE FCode='Ecp.ProcessKnowledge.cbmLiteSettings'");

    PreparedStatement ps = cn.prepareStatement(
        "INSERT INTO TsPage(FId,FName,FTitle,FCode,FType,FUrl,FActionMethodName,FUnitId,FVisible,FIsSlavePage,FListMultiRow)" +
        " VALUES(?,?,?,?,?,?,?,?,1,0,0)");
    ps.setString(1, pageId);
    ps.setString(2, "CBM-Lite 設定"); ps.setString(3, "CBM-Lite 設定");
    ps.setString(4, "Ecp.ProcessKnowledge.cbmLiteSettings");
    ps.setString(5, "Other");
    ps.setString(6, "ecp/page/cbmlite/CbmLiteSettings.jsp");
    ps.setString(7, "Ecp.ProcessKnowledge.cbmLitePrepareSettings");
    ps.setString(8, "fd65ef8a-17e8-4096-afb0-8c2d459b1e5c");
    ps.executeUpdate();

    cn.createStatement().executeUpdate(
        "INSERT INTO TsMenu(FId,FParentId,FIndex,FName,FType,FPageId,FLicensed,FEnabled,FIsSystemLevel,FReplaceByChildren,FBuiltin)" +
        " VALUES('" + menuId + "','00000000-0000-0000-0008-990000000020',13,'CBM-Lite 設定','InternalPage','" + pageId + "',1,1,0,0,0)");

    // ← 必須補這一步
    cn.createStatement().executeUpdate(
        "UPDATE TsMenu SET FTreeLevel=3,FTreeSerial='004.002.013'," +
        "FTabTitleSource=NULL,FSubMenuSource=NULL,FHideInMainMenu=NULL" +
        " WHERE FName='CBM-Lite 設定'");
}
```

## HTTP Endpoints（CbmLiteServer.java 全部端點）

| Path | Method | 說明 |
|------|--------|------|
| `/health` | GET | 健康檢查 |
| `/cbm/ask` | - | 知識庫問答 |
| `/cbm/search` | - | 知識庫搜尋 |
| `/gateway`、`/line/webhook` | POST | LINE webhook 入口 |
| `/agent` | GET | 文字客服台 HTML |
| `/agent/rooms` | GET | 進線中聊天室清單（含 `closed` 欄位，也列近 24h 內關閉的） |
| `/agent/messages?roomId=` | GET | 某房間對話 |
| `/agent/send` | POST | `{roomId, content}` 客服回覆，立即 push 到 LINE |
| `/agent/close` | POST | `{roomId}` 結束服務，push 結束訊息給 user |
| `/liff`、`/liff/verify` | GET/POST | LIFF 頁面與 idToken 驗證 |
| `/admin/reload` | POST | Hot reload config |
| `/MediaProxy` | - | 媒體代理 |

> `com.sun.net.httpserver` 是最長前綴匹配：`/agent/rooms` 等具體路徑需在 `/agent` **之前**註冊，否則會被 `/agent` 的 handler 攔截。

## 回答優先順序

1. 關鍵字規則（`cbm-lite-replies.conf`，每次請求重讀，不需重啟）
2. 知識庫搜尋（Obsidian or DB，sliding-window CJK，score ≥ `cbm.lite.kb.minScore`）
3. LLM 備援（`escalatePrompt` 為 system prompt）

## LINE Webhook 整合

### 現行 Webhook URL 與路由

實際登記在 LINE 後台的 Webhook URL、以及各 hostname 怎麼分流到 cbm-lite（12621）still，以 `cloudflare-tunnel` skill 的「Current Ingress」章節為準（那邊有完整 config.yml），本 skill 不重複貼一份會過期的副本。重點只記：cbm-lite 本身固定監聽 `127.0.0.1:12621`，對外靠 Cloudflare 依 hostname/path 轉發過來，跟 Tomcat/ECP 是分開的兩個服務。

### Async Message Flow

```
LINE user 傳訊息
  → LINE Platform POST 到 webhook URL
  → Cloudflare Tunnel → 127.0.0.1:12621/gateway
  → HMAC-SHA256 signature 驗證
  → webhook 立即回 HTTP 200（~50ms）
  → msgExecutor thread: lineAnswer(userId, text)
      ├─ 「真人客服」     → insertAiEvent("transfer") + switchToAgentMode()
      ├─ 已在 agent 模式  → forwardToAgentRoom()
      ├─ 「👍」           → updateAiFeedback(1)
      ├─ 「👎」           → updateAiFeedback(0)
      ├─ keywordReply()   → lineFlexWithFeedback(answer)
      ├─ KB score ≥ min   → lineFlexWithFeedback(answer)
      └─ LLM              → lineFlexWithFeedback(answer)
  → pushLine(userId, messages)
      → POST https://api.line.me/v2/bot/message/push
```

**關鍵：用 `pushLine()` 不是 `replyLine()`**。replyToken 一次性且有時效，非同步情境下必掛。

### LINE Event Handling

| Event type | 處理 |
|-----------|------|
| `message`（text） | 非同步 `lineAnswer()` → push 回應 |
| `follow` | 連結 richMenu + reply 歡迎訊息 |
| `unfollow` | 記錄事件；若在 agent 模式則自動呼叫 `endAgentMode()` 關閉聊天室（只在封鎖/取消追蹤時觸發，單純離開 chat 不發事件） |
| `postback` data=`end-agent` | 結束真人客服模式 |

### Flex Feedback Message — lineFlexWithFeedback()

所有 AI/KB/keyword 回覆都用一個 Flex Message bubble，右下角附 👎👍 按鈕：

```java
private static JSONArray lineFlexWithFeedback(String text) {
    // body: text, size "lg", wrap
    // footer: filler + 👎 button + 👍 button (right-aligned)
    // sender: botName + botIconUrl (fetched at startup from /v2/bot/info)
}
```

**Quick Reply 陷阱：** 用空白字元（如全形空白 U+3000）當 spacer 標籤會讓 LINE Push API 回錯誤 `` `label` must be specified ``。`lineTextWithQR()` 現在會跳過 `label.trim().isEmpty()` 的標籤。

### Bot Profile 自動抓取

啟動時呼叫 LINE `GET /v2/bot/info`，快取 `botIconUrl`／`botName` 用於 Flex messages 的 sender 欄位。若 OA 無大頭貼 → log 出現 `icon=none`，sender icon 省略；要修就去 LINE OA Manager → Account Settings → Basic Settings 上傳，然後重啟 cbm-lite。

### 簽章驗證

`X-Line-Signature: Base64(HMAC-SHA256(channelSecret, rawBody))`。若 `channelSecret` 在設定裡是空的，驗證會被跳過（僅限開發用）。

### LINE Developers Console 設定步驟

1. 建立 Messaging API channel
2. Webhook URL 填實際登記的那個（見 `cloudflare-tunnel` skill）
3. 開啟 **Use webhook**
4. 複製 **Channel secret** + **Channel access token (long-lived)** → 填進 `cbm-lite.properties`
5. 按 **Verify** — 應回 Success

### 關鍵字規則的陷阱

Keywords 是 `contains` 比對——太寬泛的關鍵字會攔截所有含該字的訊息。

**壞例子：** `時間 = ...` → 連「上班時間」「客服時間」都被攔下  
**好例子：** 只放不需要查知識庫的招呼語/固定用語（`你好`、`謝謝`）

## AI ↔ 真人客服模式切換（核心邏輯）

所有相關程式在 CbmLiteServer.java（見文首路徑說明）。客服操作畫面見下方「文字客服台」章節。

### 兩種模式

| 模式 | 行為 | 回覆格式 |
|------|------|---------|
| `ai`（預設） | 關鍵詞回覆 → 知識庫 → AI 判斷 | **Flex 卡片** + 右下角 👎👍 回饋按鈕 |
| `agent`（真人） | 每句訊息寫入 `TcChatMessage`，客服在客服台回覆；不經 AI | **純文字**，無按鈕 |

> 設計原則：AI 回覆需要品質回饋（👎👍），真人客服是一般對話不需要。

狀態存在記憶體 `ConcurrentHashMap<String,UserState> userStates` + DB 表
`CbmLiteUserState`（cbm-lite 重啟後 lazy 從 DB 還原）。

### 路由：`lineAnswer(userId, text, replyToken)`

```
1. isAgentRequest(text)  → 含「真人/客服/轉接/人工」→ switchToAgentMode()   直接轉人工
2. 若 state.mode == "agent":
     isEndAgentRequest(text) → 含「結束/離開/掰/bye/exit/回ai」→ endAgentMode()  回 AI
     否則 → forwardToAgentRoom()  寫入 TcChatMessage(FCategory='Outer')
3. AI 模式:
     keywordReply()           → cbm-lite-replies.conf 命中即回
     知識庫 answerQuestion(text, in, false)
            source=knowledge 且 score >= cbm.lite.kb.minScore(50) → 用知識庫答案
     llmAnswerOrEscalate(text)
            AI 能答 → 回 AI 答案
            AI 回 __ESCALATE__ / 空 / LLM 關閉 → 自動 switchToAgentMode()  轉人工
```

### 進入真人客服的途徑

1. **關鍵詞**：客戶發「真人客服 / 轉接 / 人工 / 真人」
2. **Rich Menu 按鈕**：主選單「真人客服」格子送出文字「真人客服」→ 同上
3. **客戶主動**：AI 說「這個我不清楚，需要真人客服嗎」後客戶自己按轉接

> `__ESCALATE__` 自動轉接機制已**停用**（從 escalatePrompt 移除）。AI 不再自動轉，改由客戶主動決定，避免閒聊/常識問題誤觸轉接。

### 知識庫信心門檻（關鍵調優點）

`answerQuestion` 內含 `searchBySubstrings` 模糊拆詞，會讓**無關問句也弱匹配**到知識庫。
範例：「我要退款訂單12345」曾誤匹配「LINE 客服營業時間」得 40 分。

**解法**：`lineAnswer` 只在 `score >= cbm.lite.kb.minScore`（目前 **30**）才採用知識庫。
- 真匹配「上班時間」→ 60 分 → 用知識庫
- 弱匹配「我要退款」→ 25 分 → 落到 AI

調門檻：改 `cbm.lite.kb.minScore`（調高=更嚴格、更多走 AI；調低=更依賴知識庫），改完**重啟 cbm-lite**。

### escalate prompt 策略

`llmAnswerOrEscalate` 使用 `cbm.lite.llm.escalatePrompt` 作為 system prompt（**不是** `systemPrompt`）。

**目前策略：AI 不自動轉人工**。移除了 `__ESCALATE__` 機制，改為：
- AI 知道答案 → 簡短回答（1-2 句）
- AI 不知道 → 說「這個我不清楚，需要真人客服嗎」，讓**客戶自己決定**是否轉接
- 客戶主動按「真人客服」按鈕或輸入關鍵詞才轉接

**原因**：`__ESCALATE__` 自動轉接太激進，連閒聊/常識問題也會誤觸，改由客戶主動選擇更合適。

目前 escalatePrompt：
```
你是客服助理，用台灣繁體中文回答，嚴守以下規則：
【回答格式】只能回一到兩句話，不用emoji，不要多說廢話。
【不知道時】若你不知道答案，直接說「這個我不清楚，需要真人客服嗎」絕對不要猜測或編造。
【可以自行回答】問候、閒聊、天氣、常識等，簡短親切回答。
```

### AI 多輪上下文記憶

AI 模式的對話**有上下文記憶且持久化**，多輪連貫、cbm-lite 重啟後仍記得：

- 每個 LINE userId 在 `UserState.aiHistory`（`List<JSONObject>`，role=user/assistant）保留最近數輪
- `recordAiTurn(state, userId, userText, aiReply)` 在每個 AI 回覆點（關鍵詞/知識庫/LLM）記錄一輪，
  並**即時 `saveUserState` 寫入 DB**
- 持久化欄位：`CbmLiteUserState.FAiHistory`（MEDIUMTEXT，存 JSON 陣列）。
  `loadUserState` 反序列化還原；`initDb` 用 `ALTER TABLE … ADD COLUMN IF NOT EXISTS` 為舊表補欄
- `llmAnswerOrEscalate(question, history)` 呼叫 LLM 時把歷史一併放進 messages
- 上限 `cbm.lite.llm.historyMaxTurns`（預設 8 輪=16 則），超過裁掉最舊的
- 轉人工途中的對話走 TcChatMessage，不進 aiHistory

驗證（**含重啟**）：問「我叫阿華養了叫旺財的狗」→ **重啟 cbm-lite** → 問「我的狗叫什麼名字」，
AI 應從 DB 還原並答出「旺財」。

### 相關設定（cbm-lite.properties）

```properties
cbm.lite.kb.minScore=30
cbm.lite.llm.historyMaxTurns=8
cbm.lite.llm.escalatePrompt=你是客服助理，用台灣繁體中文回答…不知道就說「這個我不清楚，需要真人客服嗎」…
cbm.lite.agent.startMessage=已為您轉接真人客服…
cbm.lite.agent.endMessage=已結束真人客服，返回AI助理模式…
cbm.lite.agent.forwardedMessage=          # 空=客戶訊息轉到客服台後不回確認訊息
cbm.lite.agent.pollIntervalMs=3000
```
程式中對每個 prompt/門檻都有 fallback 預設值，properties 缺鍵也能跑；改 properties 後**重啟 cbm-lite** 才生效。

### 相關 DB 表（皆已存在）

- `CbmLiteUserState` — cbm-lite 自建：`FLineUserId, FMode, FChatRoomId, FChatManId, FLastAgentMsgMs, FAiHistory(JSON)`
- `TcChatRoom` — 聊天室：`FChatId`=LINE userId、`FIsService`=1、`FCloseTime` NULL=進線中。
  `FName`/`FCreateManName`=客服台顯示名。`createChatRoom` 會先 call `lineProfileName(userId)`
  （`GET https://api.line.me/v2/bot/profile/{userId}`，需 channelAccessToken）取 LINE `displayName`
  當房名；取不到才 fallback `LINE:<userId前8碼>`。**只在建房時抓一次**，既有房間不會回填。
- `TcChatMessage` — 訊息：`FCategory`='Outer'（客戶）/'Inner'（客服）
- `TcChatMan` / `TcChatRoomMember` — 對話人/成員

### 回推客服訊息到 LINE

背景執行緒 `startAgentReplyPoller`（每 3 秒）+ 客服台 `agentSend` 主動推送雙保險：
查 `TcChatMessage WHERE FCategory='Inner' AND FSendTime > lastAgentMsgMs` → `pushLine(userId, content)`
（LINE Push API，需 channelAccessToken）。送出後推進 `lastAgentMsgMs` 防重複推送。

### 同一人不論 AI/轉真人/退出真人都要在同一筆聊天紀錄（曾修過的 bug）

**曾經的 bug**：`endAgentMode()`（客戶輸入「結束服務」退出真人客服時呼叫）會把 `TcChatRoom.FCloseTime` 設為 `NOW()` 並清空 `state.chatRoomId`。而每則進線訊息（不分 AI/真人模式）都會呼叫 `recordIncomingLineMessage()` → 找「未關閉」的房間，找不到就新建一筆。房間一關閉，下一句話（不管是繼續跟 AI 聊或再次轉真人）都會被拆到一筆新紀錄，造成同一人被拆成好幾筆對話。

**修法**：`endAgentMode()` 現在只切換 `state.mode` 回 `"ai"`，**不再**關閉房間、不清空 `chatRoomId`。同一位 LINE 使用者從第一句話開始，不論 AI 對話、轉真人、退出真人，全部停留在同一筆 `TcChatRoom`，客服台看到的紀錄才會連續完整。（`/agent/close` 手動關閉房間的功能不受影響，那是客服主動結束整個服務的操作，跟自動退出真人模式是兩回事。）

### 測試（PowerShell）

```powershell
$secret = (Get-Content "<cbm-lite.properties 路徑>" | Select-String "channelSecret=").ToString().Split("=",2)[1].Trim()
function Ask($text,$uid){
  $body = "{`"events`":[{`"type`":`"message`",`"replyToken`":`"rt1`",`"source`":{`"userId`":`"$uid`",`"type`":`"user`"},`"message`":{`"type`":`"text`",`"id`":`"1`",`"text`":`"$text`"}}]}"
  $hmac = [System.Security.Cryptography.HMACSHA256]::new([System.Text.Encoding]::UTF8.GetBytes($secret))
  $sig  = [Convert]::ToBase64String($hmac.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($body)))
  (Invoke-RestMethod "http://127.0.0.1:12621/gateway" -Method POST -Body ([Text.Encoding]::UTF8.GetBytes($body)) -Headers @{"x-line-signature"=$sig;"Content-Type"="application/json; charset=utf-8"} -TimeoutSec 40).replies[0].text
}
Ask '真人客服' 'Utest1'      # 轉人工
Ask '上班時間' 'Utest2'      # 知識庫
Ask '端午要去台東玩' 'Utest3' # AI 親切回答（不轉）
Ask '我要退款' 'Utest4'      # 轉人工
```

## 文字客服台（Agent Console，`/agent`）

真人客服在瀏覽器操作的畫面，cbm-lite 直接 serve（不依賴 ECP 前端框架）。

### 左側房間列表行為
- 顯示**進行中**（開放）及**最近 24 小時內關閉**的對話
- 開放中：正常樣式，有未讀計數藍色標籤
- 已關閉：半透明灰色，顯示灰色「已關閉」標籤；仍可點選查看歷史（唯讀）
- 開放房間排前面，關閉房間排後面，各自按最後訊息時間降序

### 右側對話框行為
- **選擇房間**：完整載入歷史，自動捲到最底
- **新訊息輪詢**（每 2 秒）：只 append 新訊息（`insertAdjacentHTML`），不重建 DOM
  - 若 scroll 在底部（距底 < 80px）→ 自動往下捲
  - 若客服已往上翻 → 保持位置不動
- **服務結束後**：輸入框 + 送出鈕反白（disabled），「結束服務」按鈕改為灰色「已結束」

### 服務結束觸發條件
| 觸發方式 | 說明 |
|---------|------|
| 客服點「結束服務」 | 立即反白，呼叫 `/agent/close` |
| user 傳「結束」「離開」等字 | `isEndAgentRequest()` → `endAgentMode()` |
| user 按 postback `end-agent` 按鈕 | webhook postback handler |
| user unfollow/封鎖 bot | LINE `unfollow` event → `endAgentMode()` |
| `loadRooms` 輪詢偵測到已關閉 | 前端每 3 秒偵測 `closed:true` → `disableComposer()` |

### 進入客服台的存取方式（曾用過的 Tomcat Proxy 架構，狀態待確認）

`hch.james-huang.org` 是走 Cloudflare **path 分流**，`^/agent`、`^/gateway`、`^/liff` 這幾條 path 直接轉去 `127.0.0.1:12621`（cbm-lite），其餘走 Tomcat 22821（ECP）——完整規則見 `cloudflare-tunnel` skill。

⚠️ **歷史架構，可能已被上面這種直接 path 分流取代**：曾經有一版做法是額外建一個獨立 Tomcat context `/cbm-proxy`（裝一支 `CbmProxyServlet`，完全繞開 Quicksilver 的 `StaticResourceFilter`），把 aipower（Tomcat）收到的 `/cbm-proxy/*` 請求轉發到 `http://127.0.0.1:12621/*`，理由是「Cloudflare 當時只通到 Tomcat，cbm-lite port 12621 從瀏覽器直接不可達」。如果現在的 Cloudflare ingress 已經有直接指到 12621 的 path 規則（見上段），這層 Tomcat 轉發多半已經不需要了；但如果之後又遇到「cbm-lite 從外部連不到、只能透過 Tomcat」的情境，可以考慮重建這個做法：

<details>
<summary>CbmProxyServlet 架構細節（點開展開）</summary>

**為何不直接在 aipower webapp 裡加 servlet**：Quicksilver 的 `StaticResourceFilter`（`quicksilver-module-main` 的 `web-fragment.xml` 宣告 `<url-pattern>/*</url-pattern>` 且排最先執行）對它不認識的路徑直接回 404，不呼叫 `chain.doFilter()`。所以在 aipower webapp 裡加任何 servlet 路徑都不通，必須是**獨立 Tomcat context**。

**目錄結構：**
```
<某目錄>\cbm-proxy-app\
└── WEB-INF\
    ├── web.xml
    └── classes\com\chainsea\ecp\cbmlite\CbmProxyServlet.class
```

**Context 描述檔**（`<chainsea>\apache-tomcat\conf\Catalina\localhost\cbm-proxy.xml`）：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context docBase="<cbm-proxy-app 目錄>" reloadable="false"/>
```

**CbmProxyServlet.java** 原始碼在 `<chainsea>\tool\src\CbmProxyServlet.java`，package `com.chainsea.ecp.cbmlite`，`TARGET = "http://127.0.0.1:12621"`，複製 request/response headers 與 body，HTTP 4xx/5xx 時回 502。

**CBM_BASE 注入機制**：`agentConsole` handler 讀 `cbm.lite.agent.proxyBase`（預設空字串），替換 `AGENT_HTML` 裡的 `__CBM_BASE__` 佔位符：
```java
String proxyBase = config.getProperty("cbm.lite.agent.proxyBase", "");
respondHtml(ex, AGENT_HTML.replace("__CBM_BASE__", proxyBase));
```
前端所有 fetch 呼叫都前綴 `CBM_BASE`（例：`fetch(CBM_BASE + '/agent/rooms')`）。`CBM_BASE=''` 時走相對路徑，本機直連 12621 仍可用。

```properties
cbm.lite.agent.proxyBase=/cbm-proxy
```

</details>

### aipower 選單設定（文字客服入口）

```sql
UPDATE TsMenu
SET FExternalPageUrl = '<客服台目前實際對外網址，見 cloudflare-tunnel skill>'
WHERE FName = '文字客服';
```

> 用 loopback 位址（如 `http://127.0.0.1:12621/agent`）從 Cloudflare 公網瀏覽器打開會被 Chrome **Private Network Access** 封鎖（公網頁面連 loopback 位址），務必填實際對外 hostname。

**三個必懂的坑（TsMenu 層級/授權）：**

1. **`FLicensed` 必須=1**。`getTree` 用 `FLicensed=1 AND FEnabled=1` 過濾；
   未授權的項目 `data.page` 為 null，前端點擊時 `eval(undefined)` **靜默失敗**。
   也要在 `TsRoleMenu` 給對應角色授權。

2. **層級**：OutlookBar 結構是 Module→Bar→Leaf。`FParentId` 必須指向某個 Directory(Bar)
   下。掛在 Module 正下方系統會主動把 `FType` 改回 `Directory`，點了只展開不開頁。

3. **外部 URL 用 `FExternalPageUrl`**。優先序：`FExternalPageUrl > FFunctionName > FPageId`。
   改完需**登出再登入**（選單樹在登入時建進 session，F5 不重建）。

### 為何不啟用完整 ECP 原生客服台

ECP 原生 Chat 後端曾與目前 base 版本不相容：共存大量同名 class 版本混用 → 執行期 `NoSuchMethodError`；且部分 servlet 依賴缺失的第三方 native library → context 起不來 → 全 404。**結論**：維持較舊的 base + 自建 cbm-lite 客服台，零風險。若日後升級到相容版本，這個限制可能已解除，值得重新評估。

### 運維

- **重啟**：先確認舊 process 真的死透（見下方「重啟坑」），再重新啟動
- **日誌**：`cbm-lite.log` / `cbm-lite.err.log`（見文首「目錄結構」的 log 陷阱）
- **健康檢查**：`GET http://127.0.0.1:12621/health`（直連）
- **cbm-lite 設定重載**：`GET http://127.0.0.1:12621/admin/reload`（不需重啟）
- 清測試資料：`UPDATE TcChatRoom SET FCloseTime=NOW() WHERE FChatId LIKE 'Utest%' AND FCloseTime IS NULL;`

## LIFF（LINE Front-end Framework）

cbm-lite 在同一 port（12621）serve 一個 LIFF web app，用在需要 userId 但純文字/Flex 不夠的場景（表單、知識庫頁面、會員中心）。**已上線運作中**。

| Path | Method | 說明 |
|------|--------|------|
| `/liff` | GET | LIFF HTML 頁（`LIFF_HTML`），serve 時把設定的 LIFF ID 注入 `liff.init()` |
| `/liff/verify` | POST | `{idToken}` → 後端打 `https://api.line.me/oauth2/v2.1/verify`（form-urlencoded）驗證，回 `{userId(sub), name, picture, email}` |

**安全關鍵：** 絕不信任前端傳來的 userId。可信 userId = verify 回傳的 `sub`。

### 已部署的實際值

| 項目 | 值 |
|------|-----|
| LIFF ID | `2010442070-e4uFbN3W` |
| LIFF URL（分享/選單用） | `https://liff.line.me/2010442070-e4uFbN3W` |
| Endpoint URL | `https://hch.james-huang.org/liff`（Cloudflare path 分流 `^/liff` → 12621，見 `cloudflare-tunnel` skill） |
| LINE Login channel | `HCH LIFF`（與 Messaging API channel 同 provider → userId 一致） |
| Scopes | openid, profile |
| Rich Menu 入口 | 子選單右上「會員中心」uri action（見 `cbm-richmenu` skill） |

### Config (cbm-lite.properties)
```properties
cbm.lite.liff.id=2010442070-e4uFbN3W
cbm.lite.liff.channelId=2010442070     # 留空也會自動取 "-" 前數字 (= verify 的 client_id)
```

### 重新建立（若日後換 channel / LIFF app）
1. LINE Developers Console → 同一個 provider 下建 **LINE Login** channel（App types 勾 Web app）
   - ⚠️ LIFF **不能**加在 Messaging API channel（LINE 已禁止，LIFF tab 沒有 Add 按鈕）
2. 該 channel → **LIFF** tab → Add：Endpoint URL = `https://hch.james-huang.org/liff`，Size Tall，Scopes 勾 `openid` `profile`
3. 複製 LIFF ID 填 `cbm.lite.liff.id`，重啟 cbm-lite
4. Rich Menu 按鈕用 uri action：`https://liff.line.me/{liffId}`（見 `cbm-richmenu` skill）

### Notes
- `httpPostForm()` 是給 verify 用的 form-urlencoded POST（`httpPost()` 只送 JSON）
- 從一般瀏覽器開啟時 `liff.login()` 會自動導向；LIFF Browser 內自動帶登入態
- 驗證失敗常見原因：token 過期、`client_id` 與 LIFF channel 不符
- 改 `cbm.lite.liff.*` 後需重啟 cbm-lite（Java 啟動讀入）

## 真人客服模式 Rich Menu：紅底「結束服務」

進入真人客服時 `switchToAgentMode()` 呼叫 `linkRichMenu(userId, "cbm.lite.line.richMenuAgentId")`，
切到一個獨立的「主選單(客服中)」1×3 選單（跟主選單同版面，只把中間格換成紅底「結束服務」），
讓使用者從選單顏色就能分辨目前是不是在真人客服模式。退出真人（`endAgentMode`）會切回
`cbm.lite.line.richMenuId`（一般主選單）。

建立/重建這個選單用 `setup-richmenu-agent.ps1`（會自動寫回
`cbm.lite.line.richMenuAgentId`）。若要連主選單/子選單一起重建，用 `setup-richmenu.ps1`
（完整版）；換 LINE channel 時搬整套選單也在同一份，細節見 `cbm-richmenu` skill。

## 關鍵字回覆規則

`cbm-lite-replies.conf` — 每次請求重讀，不需重啟：

```
__welcome__ = 歡迎，請直接輸入您的問題。
__default__ =
你好 = 您好，請問有什麼可以協助您的？
謝謝 = 不客氣。
```

- `__default__` 留空 = 交給知識庫/LLM
- 值可用 `\n` 換行（程式碼 `.replace("\\n", "\n")`）

## Bot Profile 自動抓取

見上方 LINE Webhook 整合章節。

## ⚠️ 目前實際指向：Lab2 + DeepSeek 直連（非 AiProxy）

`cbm.lite.db.*`／`cbm.lite.llm.*` 曾被改成指向 Lab2 那套部署（見 `ecp-lab2-instance` skill）而非原本的舊部署，且 LLM 端點**跳過 AiProxy、直接打 DeepSeek**：

```properties
cbm.lite.db.url=jdbc:mariadb://localhost:3306/default?useSSL=false&characterEncoding=UTF-8
cbm.lite.llm.url=https://api.deepseek.com/chat/completions   # 不是 /aipower/v1/chat/completions
cbm.lite.llm.model=deepseek-chat
cbm.lite.llm.apiKey=sk-...（DeepSeek key，見 ecp-deepseek skill）
```

**副作用**：跳過 AiProxy 就等於跳過 `AiProxyLlmRunnerImpl` 的 tool-calling loop（`search_knowledge_base`/`query_database`）——AI 建議回覆會是**純對話**，不會查知識庫/CRM。若要恢復工具呼叫，要嘛切回走 AiProxy 的 `/aipower/v1/chat/completions`（但目標環境要真的有裝 AiProxy），要嘛在 cbm-lite 自己實作一份 client-side tool-calling loop。

**LINE channel secret 要跟 LINE Developers Console 上顯示的值完全一致**——這裡的密鑰是每個 channel 各自的，若 webhook 一直 403/連不上，先去 LINE 後台核對 Channel secret / Channel access token 是否跟這裡的設定相符（可能是密鑰被重新產生過）。

## Config 參考（完整，可用於災難復原；以下欄位名稱/結構通用，實際部署路徑值需現場確認）

```properties
cbm.lite.server.host=0.0.0.0
cbm.lite.server.port=12621
cbm.lite.server.threads=8
cbm.lite.strictMatch=false
cbm.lite.http.timeoutMs=15000
cbm.lite.kb.source=obsidian
cbm.lite.kb.minScore=30
cbm.lite.obsidian.url=http://127.0.0.1:27123
cbm.lite.obsidian.apiKey=<hex key>
cbm.lite.obsidian.excludePrefix=copilot/
cbm.lite.obsidian.cacheTtlSecs=300
cbm.lite.db.url=jdbc:mariadb://localhost:3306/default?useSSL=false&characterEncoding=UTF-8
cbm.lite.db.port=3306
cbm.lite.db.name=default
cbm.lite.llm.enabled=true
cbm.lite.llm.url=http://127.0.0.1:22821/aipower/v1/chat/completions
cbm.lite.llm.model=ecp-ai
cbm.lite.llm.apiKey=local
cbm.lite.llm.systemPrompt=<single line>
cbm.lite.llm.escalatePrompt=<single line>
cbm.lite.llm.historyMaxTurns=8
cbm.lite.line.enabled=true
cbm.lite.line.channelSecret=<secret>
cbm.lite.line.channelAccessToken=<secret>
cbm.lite.line.repliesFile=<路徑>\cbm-lite-replies.conf
cbm.lite.line.welcomeMessage=您好！\n若需要真人客服，請輸入「真人客服」。
cbm.lite.line.richMenuId=richmenu-...
cbm.lite.line.richMenuSubId=richmenu-...
cbm.lite.line.richMenuSub2Id=richmenu-...
cbm.lite.line.richMenuAgentId=richmenu-...   # 真人客服模式用的紅底「結束服務」選單
cbm.lite.agent.startMessage=已為您轉接真人客服。\n如需返回AI助理，請輸入「結束服務」。
cbm.lite.agent.endMessage=已結束真人客服，返回AI助理模式。
cbm.lite.agent.alreadyAgentMessage=您目前已在真人客服模式。\n如需返回AI助理，請輸入「結束服務」。
cbm.lite.agent.errorMessage=抱歉，目前無法轉接真人客服，請稍後再試。
cbm.lite.agent.forwardedMessage=
cbm.lite.agent.pollIntervalMs=3000
cbm.lite.liff.id=2010442070-e4uFbN3W
cbm.lite.liff.channelId=2010442070
```

注意：多行訊息用 `\n`（兩個字元：反斜線 + n），**不是**實際換行。

## 已知問題 / 坑

### 坑：舊 process 殺不掉
若 cbm-lite 是由高權限腳本啟動，PowerShell `Stop-Process` 及 `taskkill` 可能 Access Denied。需在用戶終端用 `! taskkill /F /PID <pid>` 或 Task Manager 手動終止。

### 坑：重啟時 BindException 誤判「掛了」

重啟 cbm-lite 若沒先確認舊進程真的死透，開的新 java 進程會在 `HttpServer.create()` 那行丟 `Exception in thread "main" java.net.BindException: Address already in use: bind` 並立刻結束——但**這個崩潰不會反映在 `/health` 的可用性上**，因為真正還在監聽 12621 的是那個沒被殺乾淨的舊進程。容易誤判成「服務掛了又活過來」或「設定沒生效」（其實是舊進程一直在用舊設定，新的改動從未真正上線）。

**正確重啟程序**：
```powershell
$p = (Get-NetTCPConnection -LocalPort 12621 -ErrorAction SilentlyContinue | Select-Object -First 1).OwningProcess
if ($p) { Stop-Process -Id $p -Force }
Start-Sleep -Seconds 2
# 用 Get-CimInstance Win32_Process 確認真的沒有 java.exe 的 CommandLine 含 CbmLiteServer 才繼續
Start-Process "<start-cbm-lite.bat 路徑>"
```
改完設定（尤其是 channelSecret 這類影響簽章驗證的值）後，**務必**用 `Get-CimInstance Win32_Process -Filter "name='java.exe'" | Select ProcessId,CreationDate,CommandLine` 核對 cbm-lite 進程的 `CreationDate` 確實在改動之後，別只看 `/health` 回 200 就以為新設定生效了。

### ⚠️ 未解決 bug：真實文字訊息 webhook 回 200 但沒有存進 TcChatMessage

**現象**：LINE Developers Console 的「Verify」按鈕成功（Success），手動用正確簽章送一個**空 events**（`{"events":[]}`）也回 200——但送一個**真實的 `message`/`text` 事件**（模擬真人發送文字），一樣回 200，訊息卻完全沒進 `TcChatMessage`、對話房間清單也不會更新。

**已排除的可能**：
- 不是 channel secret 錯誤（用當前設定值算出的簽章驗證通過）。
- 不是 DB 連線問題（`/agent/rooms`、`/agent/suggest` 直接查詢都正常）。
- 不是 cbm-lite 掛掉（進程持續存活，`/health` 全程 200）。
- 不是路由問題（本機直連 `127.0.0.1:12621` 與走公開網域結果一致）。

**重現方式**：
```python
import hmac, hashlib, base64, json, requests, time
secret = "<cbm.lite.line.channelSecret 目前值>"
body = json.dumps({
    "destination": "U-test-destination",
    "events": [{
        "type": "message",
        "message": {"type": "text", "id": "999999", "text": "測試訊息" + str(int(time.time()))},
        "timestamp": int(time.time()*1000),
        "source": {"type": "user", "userId": "<真實或測試 userId>"},
        "replyToken": "dummy-reply-token",
        "mode": "active"
    }]
}, ensure_ascii=False)
sig = base64.b64encode(hmac.new(secret.encode(), body.encode('utf-8'), hashlib.sha256).digest()).decode()
r = requests.post("<實際 webhook URL>", data=body.encode('utf-8'),
                   headers={"Content-Type":"application/json", "X-Line-Signature": sig}, timeout=15)
print(r.status_code, r.text)   # 200 success，但 DB 查不到這則訊息
```

**下一步該查的地方**（`CbmLiteServer.java`）：
`lineWebhook()` → 對 `"text".equals(msgType)` 的分支把處理丟進 `msgExecutor.submit(() -> { ... lineAnswer() ... pushLine() ... })` 非同步執行，且用 `try/catch` 吞掉例外只 `log.warning(...)`。**目前已知的可疑點**：
1. `lineAnswer()` 本身可能只負責「生成 AI 回覆」，從未呼叫任何寫入 `TcChatMessage`／`TcChatRoom` 的邏輯——真正負責把**來訊本身**存檔的程式碼可能在別的地方（或根本沒實作），需要追蹤 `lineAnswer` 內部呼叫鏈，確認有沒有 `INSERT INTO TcChatMessage` 這段。
2. `msgExecutor` 是非同步執行緒池，例外被 `catch (Exception e) { log.warning(...) }` 吞掉——若真的丟了例外，訊息會出現在**當次啟動的 stdout/stderr 重導向 log**（不是 `cbm-lite.log`，那個檔案的 `java.util.logging` FileHandler 疑似很久沒更新/沒 flush），務必用手動 `-RedirectStandardError` 導出的 log 才看得到當次真實輸出。
3. 檢查 `verifyLineSignature`／DB 寫入是否有 tenant/scope 過濾條件（`FTenantCode`/`FCBMTenantCode` 之類欄位）導致 INSERT 被悄悄導去別的地方或被過濾掉。

---

## Conformance Addendum

## When to Use
cbm-lite LINE 客服系統總覽——standalone server（CbmLiteServer.java，port 12621）的編譯部署、設定、DB 注冊陷阱、LINE webhook 整合、AI↔真人客服模式切換邏輯、文字客服台、LIFF。合併自原本分散的 cbm-line／cbm-line-agent／cbm-agent-console 四個 skill。

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
