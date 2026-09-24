---
name: aipower-knowledge-obsidian
description: 「aipower 知識庫」功能——Obsidian 純 UI 外掛（.obsidian/plugins/aipower-knowledge）+ aipower Docker 後端（Ecp.KnowledgeEntry/Ecp.AiProviderConfig 兩個標準六大核心 Unit，走 Bearer Token 的 LocalApi）。涵蓋知識庫 CRUD、AI 摘要/整理、跨知識庫問答（關鍵字 RAG）、可切換多組 AI 服務設定。當使用者要求「改知識庫外掛」「知識庫功能有 bug」「AI 摘要沒反應」「知識庫問答」「新增/調整 AI 服務設定」「知識庫登入失敗」，或提到 aipower-knowledge 這個外掛時使用。後端部署共用 aipower-docker-local skill 的工具（deploy-servlet.sh），前端測試共用 control-obsidian-cdp skill。
---

# aipower 知識庫（Obsidian 純 UI + aipower 後端）

## 架構

```
Obsidian 外掛（純 UI，TypeScript，D:\OB\.obsidian\plugins\aipower-knowledge）
   │  HTTPS，Authorization: Bearer <tokenId>
   ▼
aipower（D:\aipower docker，/aipower/openapi/*，TsLocalApi 反射派工）
   │
   ├─ Ecp.KnowledgeEntry（六大核心 Unit）──JDBC──▶ TcKnowledgeEntry（知識庫資料）
   ├─ Ecp.AiProviderConfig（六大核心 Unit）──JDBC──▶ TcAiProviderConfig（AI 服務設定，可存多組）
   └─ AiProviderHome.callChat()：讀目前生效的 Provider，依 FApiFormat 打
      OpenAI-Compatible 或 Anthropic 格式的外部/自架 AI API
```

**Obsidian 端完全不存資料**——vault 裡沒有任何知識庫內容，全部即時打 API 讀寫 aipower 的
MariaDB。這是使用者明確選定的方向（沿用 aipower 既有的六大核心＋LocalApi Bearer Token 機制，
不是用 frontmatter/Dataview/Bases 那條路）。

## 檔案位置

- **後端原始碼**：`C:\Users\HCH\.claude\skills\aipower-docker-local\knowledge-integration\`
  （`src/com/chainsea/ecp/knowledge/`、`src/com/chainsea/ecp/aiprovider/`、
  `knowledge_unit.sql`）——照 `asset-integration/` 的六大核心模板寫的，改動/新增後端邏輯要去
  這裡改原始碼，用同目錄樹狀結構（Home/Model/Dao/Service/Action/Api 分層）。
- **Obsidian 外掛原始碼**：`D:\OB\.obsidian\plugins\aipower-knowledge\src\`
  （`main.ts`、`settings.ts`、`api/client.ts`、`views/KnowledgeListView.ts`、
  `modals/EntryFormModal.ts`、`modals/AskModal.ts`、`modals/ProviderFormModal.ts`）。
  直接在這個路徑下開發（Obsidian 社群外掛常見做法，不用另外複製一份）。

## 後端 Unit / API 摘要

| Unit | 表 | LocalApi path | 說明 |
|---|---|---|---|
| `Ecp.KnowledgeEntry` | `TcKnowledgeEntry` | `knowledge/api` | 欄位：FTitle/FContent/FCategory/FTags/FSource/FSummary |
| `Ecp.AiProviderConfig` | `TcAiProviderConfig` | `ai-provider/api` | 欄位：FProviderName/FApiFormat/FBaseUrl/FApiKey/FModel/FIsActive/FDescription |
| （通用） | — | `auth/token` | 登入換 Bearer token / 撤銷，`LocalApiAuthHandler`，跟 Asset 那組完全共用同一份程式碼模式 |

`knowledge/api` 的 op：`item`/`list`/`create`/`update`/`delete`（標準 CRUD，直接包
`KnowledgeEntryApiImpl`）+ `summarize`（`{id}` 或 `{content}` → `{summary,tags[]}`，**不直接寫回
資料庫**，human-in-the-loop）+ `ask`（`{question}` → `{answer,sources[]}`，關鍵字比對找相關筆記
當上下文，非向量嵌入）。

`ai-provider/api` 的 op：標準 CRUD + `activate`（`{id}` → 單選設成使用中，透過
`AiProviderHome.setActive()`，內部用 `EntityService.getItem/update()`，**不是**直接 JDBC UPDATE
——見下方「鐵則」）。

`FApiKey` 刻意不放進 `TsListField`，`list` 回應不含這個欄位，`item`/編輯表單才看得到（比照
`TcAiProxySetting` 既有慣例）。

## 部署後端改動

跟 `aipower-docker-local` skill 的所有其他自訂整合完全同一套流程：

```bash
cd "C:\Users\HCH\.claude\skills\aipower-docker-local"
bash deploy-servlet.sh knowledge-integration/src/com/chainsea/ecp/aiprovider/AiProviderHome.java \
  knowledge-integration/src/com/chainsea/ecp/knowledge/integration/KnowledgeLocalApiHandler.java
  # 可以一次丟多個檔案，腳本會自動算出容器內路徑、編譯、部署、重啟
```

改 DB metadata（新增欄位、選單等）要改 `knowledge_unit.sql` 再重新整份執行（腳本本身是冪等的，
開頭有 DELETE 段）。

⚠ **鐵則（見 memory `feedback_aipower_entityservice_mandatory`）：任何「寫入」都要透過
`XxxHome.getService().create/update/delete(ctx,...)`，不要手刻 JDBC UPDATE/INSERT/DELETE**——
`AiProviderHome.setActive()` 一開始就是踩這個坑寫的（直接兩條 JDBC UPDATE），繞過了
`TsRolePrivilege` 權限檢查，後來改成先用純讀 JDBC 找出目前 active 的列 id（讀不受此限），
再對每一筆呼叫 `getService().getItem()/update()` 才修正。「讀」（`loadActiveConfig()`、
`KnowledgeLocalApiHandler` 的關鍵字搜尋）不受此限，繼續手寫 SQL 沒關係。

## 重建 / 重載 Obsidian 外掛

```bash
cd "D:/OB/.obsidian/plugins/aipower-knowledge"
npx tsc --noEmit   # 型別檢查
npm run build      # esbuild 產生 main.js
```

改完程式碼後用 `control-obsidian-cdp` skill 操作/驗證真實畫面（見該 skill）。**熱重載外掛不用
整個重啟 Obsidian**，用 CDP 跑：
```js
(async function() {
  const id = app.plugins.plugins['aipower-knowledge'].manifest.id;
  await app.plugins.disablePlugin(id);
  await app.plugins.enablePlugin(id);
  return 'reloaded';
})()
```
（已驗證過重載後 `KnowledgeListView` 的分頁/資料不會遺失。）第一次啟用需要使用者在
Obsidian「設定 → 社群外掛」手動打開一次（`community-plugins.json` 已經把
`aipower-knowledge` 加進去了）。

## 已知踩過的坑（都已修好，記錄原因避免以後重踩）

### 1. Java HttpClient 對 vLLM 送出的 request body 消失（HTTP/2 h2c 升級）

`AiProviderHome` 呼叫外部/自架 AI API 用的 `java.net.http.HttpClient`，**預設會對明碼
`http://` 連線嘗試 h2c（HTTP/2 cleartext）升級**。vLLM 的 uvicorn/FastAPI 伺服器不支援這個
升級，結果連線「看起來成功」但 body 完全沒送到，vLLM 回
`400 {"error":{"message":"Field required...body: None"}}`——完全看不出是協定版本問題，容易誤
以為是 JSON 組錯或認證失敗。

**修法**：`HttpClient.newBuilder().version(HttpClient.Version.HTTP_1_1)...`，明確指定
HTTP/1.1，不讓它自動嘗試升級。**任何用 Java HttpClient 打「非 HTTPS 標準雲端 API」的自架/
內網服務（vLLM、其他本地推論伺服器等）都可能踩到同一個坑**，這條經驗不限這個功能。

### 2. AI 摘要按鈕在「編輯既有條目」時把剛產生的結果自己蓋掉

`EntryFormModal` 摘要按鈕原本在成功後呼叫 `this.onOpen()` 重繪整個表單（圖方便，順便更新
標籤欄）。但 `onOpen()` 對「已存在的條目」（`this.existing?.FId` 有值）一開頭就會呼叫
`getKnowledge()` 重新從伺服器抓資料——這時摘要/標籤根本還沒按儲存，伺服器上還是空的，
重繪就把剛產生、還沒存的內容用伺服器的空值蓋掉了。**新增條目**（`existing` 是 `null`）不會
觸發這段重抓，所以只有編輯既有條目才會踩到，容易在測試時漏掉（用「+新增」測不出來）。

**修法**：拿掉 `this.onOpen()`，改成直接用原生 property setter + `dispatchEvent(new
Event('input'))` 更新摘要欄位和標籤欄位各自的 DOM 值，不重繪整個表單。**教訓：任何「表單
局部更新」都不要圖方便呼叫會重新抓資料的完整重繪函式，尤其該函式對「新建」和「編輯」兩種
模式行為不同時。**

### 3. Token 過期時完全沒有提示，設定頁還誤顯示已登入

一開始 token 失效只會拋出原始錯誤碼（如 `E.Knowledge.Token.Invalid`）直接顯示給使用者，
且 `plugin.settings.tokenId` 不會被清掉，設定頁繼續顯示「已登入」，使用者完全不知道要
重新登入。

**修法**：`AipowerClient.request()` 偵測到 token 相關錯誤碼（`Client.Token.Missing` 或
`*.Token.Invalid`/`*.Token.Required`）時自動清掉 `tokenId` 並觸發 `onTokenChanged(null)`；
新增 `formatApiError()` 統一把 token 錯誤轉成「登入已過期或尚未登入，請至外掛設定頁重新
登入」這句固定訊息（不再顯示原始錯誤碼）；`KnowledgeListView` 讀取失敗且判斷是 token
問題時，額外顯示一顆「前往設定重新登入」按鈕直接跳轉。

### 4. vLLM 的模型名稱大小寫敏感

`TcAiProviderConfig.FModel` 若跟 `GET /v1/models` 回傳的 `id` 大小寫不一致（例如存
`Qwythos-9B-v2-AWQ` 但服務端實際是 `qwythos-9b-v2-awq`），會得到
`404 model does not exist`。新增/編輯 provider 設定時，`FModel` 建議直接照抄
`curl http://<host>:<port>/v1/models` 回應裡的 `id` 值，不要憑印象輸入。

## 驗證方式

- **後端獨立驗證**：`auth/token?op=apply` 換 token → `ai-provider/api` CRUD+activate →
  `knowledge/api` CRUD+summarize+ask → 未帶 token 確認 401，全部可以直接用 curl 跑（不需要
  Obsidian），範例序列見 git 歷史/對話紀錄，或直接照抄 `asset-integration` 那次的驗證手法。
- **前端／真實 UI 驗證**：一律用 `control-obsidian-cdp` skill，不要用截圖以外的方式猜測
  Obsidian 畫面行為——目前已知的兩個真正的 bug（上面第 2、3 點）都是在自動化跑過一輪、或
  使用者實際回報後才抓到的，光看程式碼容易漏掉「編輯 vs 新增模式行為不同」這類分支。

## 測試帳密 / 已知外部依賴

- aipower 登入：`administrator` / `<ECP_PASSWORD>`（這組密碼有過變動歷史，見
  memory `feedback_never_reset_shared_aipower_password`——**不要自己重設，要用先問使用者**）。
- 目前設定好的 AI Provider：「Qwythos」，`FApiFormat=OpenAI-Compatible`，
  `FBaseUrl=http://10.145.119.19:8182/v1`，這是使用者自架的 vLLM（見 `vllm-qwythos-deploy`
  skill），**不受這個 Docker 環境對 `api.deepseek.com`/`api.openai.com`/`api.anthropic.com`
  的 DNS 封鎖影響**（那幾個網域在這個環境會被導向 `127.185.x.x` loopback，是網路層級封鎖，
  非本功能程式碼問題；自架/內網 AI 服務不受影響，這也是為什麼優先幫使用者接自架服務而不是
  硬解那個 DNS 封鎖）。

## 相關 Skill

- `aipower-docker-local`：aipower Docker 環境本身、`deploy-servlet.sh`、六大核心開發規範、
  這個環境的完整歷史脈絡。
- `control-obsidian-cdp`：用 CDP 直接操作/測試 Obsidian 真實畫面，這是使用者指定的固定測試
  方式，改這個外掛之後一律用這個驗證，不要只看程式碼猜。
- `vllm-qwythos-deploy`：Qwythos vLLM 自架部署本身（換模型、調參數）。

---

## Conformance Addendum

## When to Use
「aipower 知識庫」功能——Obsidian 純 UI 外掛（.obsidian/plugins/aipower-knowledge）+ aipower Docker 後端（Ecp.KnowledgeEntry/Ecp.AiProviderConfig 兩個標準六大核心 Unit，走 Bearer Token 的 LocalApi）。涵蓋知識庫 CRUD、AI 摘要/整理、跨知識庫問答（關鍵字 RAG）、可切換多組 AI 服務設定。當使用者要求「改知識庫外掛」「知識庫功能有 bug」「AI 摘要沒反應」「知識庫問答」「新增/調整 AI 服務設定」「知識庫登入失敗」，或提到 aipower-knowledge 這個外掛時使用。後端部署共用 aipower-docker-local skill 的工具（deploy-servlet.sh），前端測試共用 control-obsidian-cdp skill。

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
