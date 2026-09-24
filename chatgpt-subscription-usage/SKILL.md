---
name: chatgpt-subscription-usage
description: 透過 Playwright 查詢 ChatGPT 網頁版訂閱方案的使用量限制與重置時間，包含 5 小時、每週、每月等動態額度。適用於使用者詢問 ChatGPT Free／Plus／Pro／Go 的訂閱用量、剩餘百分比、重置時間，或要求確認 ChatGPT「使用情況」頁面的資料；不適用於 OpenAI API token 用量、組織成本或 API rate limit。
---

# ChatGPT 訂閱用量查詢

## 目的

查詢 ChatGPT 網頁版帳號在「設定 → 使用量」顯示的訂閱方案額度，包括：

- 方案類型
- 使用量是否達到限制
- 主要使用窗口（可能是 5 小時、每週或每月）
- 次要使用窗口
- 已使用百分比與剩餘百分比
- 重置倒數與重置時間
- 點數／credits 狀態

這不是 OpenAI API 的用量查詢。以下端點不代表 ChatGPT 訂閱額度：

- `/v1/organization/usage/...`
- `/v1/organization/costs`
- API response 的 `x-ratelimit-*` headers

## 查詢方式

優先使用 Playwright，在已登入的 ChatGPT 網頁工作階段中執行同源請求：

```javascript
fetch("/backend-api/wham/usage", {
  credentials: "include"
})
```

這個 `/backend-api/wham/usage` 是 ChatGPT 網頁使用的非公開內部端點，不是承諾穩定的公開 API。它可能改名、改變 JSON 結構或失效；不得把它描述成官方公開 API。

可同時觀察使用量設定頁面觸發的相關請求：

```text
/backend-api/pageConfigs/usage_limits
/backend-api/wham/usage
/backend-api/wham/rate-limit-reset-credits
```

其中真正包含帳號限額數值的主要資料通常來自 `/backend-api/wham/usage`；`pageConfigs/usage_limits` 主要是判斷使用量頁是否顯示。

## Playwright 流程

1. 開啟或接管 `https://chatgpt.com/`。
2. 確認目前登入的是使用者要求查詢的帳號；不要假設帳號或方案。
3. 若進入 `auth.openai.com`、登入頁或 MFA 頁面：
   - 停止自動操作。
   - 請使用者在可見瀏覽器中自行完成登入或 MFA。
   - 不要要求使用者把一次性驗證碼、Cookie 或 token 傳給 agent。
   - 使用者確認完成後，才在同一個瀏覽器工作階段繼續一次。
4. 可開啟 `https://chatgpt.com/#settings/Usage`，等待「使用量」面板載入。
5. 透過同一頁面的 `page.evaluate()` 呼叫 `/backend-api/wham/usage`。
6. 記錄 HTTP 狀態碼；只有 `200` 才解析 JSON。
7. 將原始欄位整理成易讀結果，不要輸出敏感認證資料。

## 回應欄位解析

常見回應結構：

```json
{
  "plan_type": "free",
  "rate_limit": {
    "allowed": false,
    "limit_reached": true,
    "primary_window": {
      "used_percent": 100,
      "limit_window_seconds": 2592000,
      "reset_after_seconds": 2507701,
      "reset_at": 1792721359
    },
    "secondary_window": null
  },
  "credits": {
    "has_credits": false,
    "unlimited": false,
    "balance": null
  }
}
```

解析規則：

- `plan_type`: 方案類型，例如 `free`；不可只依頁面文字猜測。
- `used_percent`: 已使用百分比。
- `remaining_percent = 100 - used_percent`。
- `limit_window_seconds`：
  - `18000` = 5 小時
  - `604800` = 每週
  - `2592000` = 30 天／每月近似窗口
  - 其他值標記為自訂窗口，不要硬套名稱。
- `reset_at`：Unix timestamp；轉換時以 `Asia/Taipei` 顯示，並保留原始 timestamp 供核對。
- `reset_after_seconds`：查詢當下的剩餘秒數；不應代替 `reset_at`。
- `secondary_window: null`：表示目前沒有第二個限額窗口，不要自行補成每週上限。
- `allowed` 與 `limit_reached`：分別回報是否允許使用及是否已達限制。
- `credits`：另外回報點數狀態，不要把點數誤當成方案訊息額度。

不同帳號／方案可能回傳不同窗口。例如：

```text
primary_window = 5 小時
secondary_window = 每週
```

也可能只有：

```text
primary_window = 每月
secondary_window = null
```

必須以本次實際回應為準。

## 安全規則

- 不要讀取、複製、列印或回傳 `Authorization`、Cookie、session token、access token、完整 Network request headers。
- 不要讓使用者把登入憑證貼到對話中。
- 不要將 ChatGPT 網頁 session token 寫入 skill、程式碼、log、Git 或 API 回應。
- 不要使用 `curl` 搭配手工複製的 Cookie／session token 作為預設流程。
- 不要把內部端點暴露成公開服務。
- 若將來要包成本機 API，預設只能綁定 `127.0.0.1`，並使用獨立且受保護的瀏覽器 profile；目前本 skill 只記錄查詢流程，不建立本機 API 服務。
- 不要透過此流程修改額度、購買點數、升級方案或執行「使用重置」。查詢必須是唯讀。

## 錯誤處理

- `401 Unauthorized`：登入狀態無效；要求使用者在可見瀏覽器中重新登入，不要索取 token。
- `403 Forbidden`：不要重試風暴；回報端點拒絕或帳號狀態，保留必要診斷資訊但不保存敏感 headers。
- `429`：停止自動重試，回報目前受到限制。
- 回應不是 JSON：回報端點格式改變，不要猜測額度。
- 找不到 `primary_window`：回報資料不完整，不要自行推導剩餘比例。
- MFA、CAPTCHA 或其他人機驗證：停止自動化，交由使用者手動完成；不得代為解題或繞過。

## 回報格式

查詢完成時，以表格簡潔回報：

| 項目 | 結果 |
|---|---|
| 查詢時間 | 台北時間 |
| 方案 | 實際 `plan_type` |
| 主要窗口 | 5 小時／每週／每月／自訂 |
| 主要窗口剩餘 | 百分比；若缺資料標記未知 |
| 主要窗口重置 | 台北時間 |
| 次要窗口 | 實際窗口或無 |
| 是否達到限制 | 是／否 |
| 點數 | 實際 credits 狀態 |

另須說明：

- 這是 ChatGPT 網頁內部端點的即時結果，不是 OpenAI API 組織用量。
- 查詢時間與資料完整度。
- 是否遇到登入、MFA、403、429 或其他驗證／阻擋。
- 若頁面資料與端點資料不一致，以實際回應為準並明確標示差異。

## 目前狀態

本 skill 目前只保存查詢規範與安全邊界，**尚未實作本機 API wrapper、常駐服務、排程查詢或 CLI 指令**。若日後要實作，必須先確認：

1. 使用哪個 Playwright 工作階段與 profile。
2. 是否採 headed persistent context。
3. 本機服務的 port、驗證方式與輸出格式。
4. 如何處理登入過期與 MFA 的人工介入。
5. 如何避免把 session 憑證寫入 log 或回應。
