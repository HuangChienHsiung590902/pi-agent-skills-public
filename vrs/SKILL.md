---
name: vrs
description: Use when operating the 手語視訊轉譯中心後台系統 (VRS) - including login, navigating to CRM or 中心工作台, clicking menu items, handling browser permission popups, or automating any workflow within sfaa-vrsp-ecp.qbicloud.com or vrs.sfaa.gov.tw
---

# VRS 手語視訊轉譯中心後台系統 操作技能

> 這是**手動 Playwright UI 操作**流程（點選、截圖辨識驗證碼）。若只是要抓資料/查報表/通話記錄，改用 API+Token 腳本流程更快，見 `vrs-service`（總覽）、`vrs-calllog`（CRM 通話記錄）、`vrs-call-export`（單通/批次通話錄音檔+逐字稿匯出）。本 skill 適用於：登入、選單導覽、或腳本流程失效時的手動操作。

## 系統概覽

本系統由兩個子系統組成，透過 SSO token 串接：

| 系統 | URL | 說明 |
|------|-----|------|
| **後台管理系統** | `https://vrs.sfaa.gov.tw/be/#/Web/Login` | 登入入口、聯絡人管理、統計報表 |
| **ECP 值機系統** | `https://sfaa-vrsp-ecp.qbicloud.com/ecp/Qs.MainFrame.page` | CRM 主系統，透過 iframe 嵌套多個頁面 |
| **中心工作台** | `https://vrs.sfaa.gov.tw/be/#/Translation/CenterDesk` | 直接跳轉後會自動 redirect 到 ECP 並嵌入於 iframe |

---

## 工具需求

**必須使用 `playwright`** 操作本系統。所有步驟使用以下工具：
- `browser_navigate` — 頁面導航
- `browser_snapshot` — 取得 a11y 樹（含 uid 供點擊用）
- `browser_take_screenshot` — 截圖確認視覺狀態
- `browser_fill_form` — 填寫表單
- `browser_type` — 輸入文字到單一欄位
- `browser_click` — 點擊元素（使用 snapshot 中的 ref）
- `browser_run_code` — 執行 JavaScript（用於複雜互動）
- `browser_wait_for` — 等待文字出現或等待時間
- `browser_press_key` — 鍵盤操作

---

## 步驟一：登入流程

### 1.1 開啟登入頁

```
browser_navigate(url="https://vrs.sfaa.gov.tw/be/#/Web/Login")
```

等待頁面載入（出現「請輸入帳號」輸入框）。

### 1.2 辨識驗證碼

登入頁有圖形驗證碼（6 位數字，扭曲+干擾線）。辨識方法、session 綁定注意事項見文末「[驗證碼辨識完整說明](#驗證碼辨識完整說明)」，這裡只走最短路徑：

```
browser_take_screenshot(element="驗證碼圖片", ref="<驗證碼ref>", filename="captcha.png")
# Claude 直接從截圖視覺辨識 6 位數字（多模態能力，不需要 opencode run/OCR）
```

### 1.3 填寫表單並登入

```
browser_fill_form(elements=[
  {"name": "帳號", "type": "textbox", "ref": "<帳號輸入框ref>", "value": "CS0006"},
  {"name": "密碼", "type": "textbox", "ref": "<密碼輸入框ref>", "value": "<ECP_PASSWORD>##"}
])
browser_type(element="驗證碼輸入框", ref="<驗證碼輸入框ref>", text="<辨識到的驗證碼>")
browser_click(element="登入按鈕", ref="<登入按鈕ref>")
```

### 1.4 確認登入成功

```
browser_wait_for(text="聯絡人管理", time=10)
```

成功後會跳轉至 `https://vrs.sfaa.gov.tw/be/#/Contact/ContactManagement`，右上角顯示使用者名稱（如「黃建雄」）。

### 常見登入元素（每次 snapshot 後確認 ref）

| 元素 | 識別方式 |
|------|---------|
| 帳號輸入框 | `textbox "請輸入帳號"` |
| 密碼輸入框 | `textbox "請輸入密碼"` |
| 驗證碼圖片 | `image "驗證碼圖片"` |
| 驗證碼輸入框 | `textbox "請輸入圖形驗證碼"` |
| 換一張圖按鈕 | `button "換一張圖"` |
| 登入按鈕 | `button "登入"` |

> ⚠️ 驗證碼辨識失敗時點「換一張圖」重試。

---

## 步驟二：處理瀏覽器原生彈窗

### 2.1 通知權限彈窗（最常見）

ECP 系統在進入後會觸發 Chrome 原生通知權限請求：
> **"sfaa-vrsp-ecp.qbicloud.com 要求下列權限：顯示通知 [允許] [封鎖]"**

Playwright **無法直接點擊** Chrome 原生 UI，改用鍵盤操作：

```
browser_press_key(key="Tab")    # 移到「允許」
browser_press_key(key="Tab")    # 移到「封鎖」
browser_press_key(key="Enter")  # 按下封鎖
```

或直接按 Escape 關閉：
```
browser_press_key(key="Escape")
```

---

## 步驟三：進入中心工作台

### 方法 A：直接導航（推薦）

```
browser_navigate(url="https://vrs.sfaa.gov.tw/be/#/Translation/CenterDesk")
browser_wait_for(text="客服工作檯", time=15)
```

系統會自動 redirect 至 ECP 並載入工作台。

### 方法 B：從後台選單點選

```javascript
// browser_run_code 找到中心工作台連結
browser_run_code(code="() => {\n  const links = document.querySelectorAll('a');\n  let found;\n  links.forEach(l => {\n    if (l.textContent.trim() === '中心工作台') found = l.href;\n  });\n  return found;\n}")
```

---

## 步驟四：進入 CRM 系統

### 4.1 點擊頂部 CRM 按鈕

```
browser_snapshot()
# 找到 "CRM" 的 ref
browser_click(element="CRM按鈕", ref="<CRM的ref>")
```

### 4.2 確認進入 CRM

```
browser_wait_for(text="工作檯", time=10)
browser_take_screenshot()
```

---

## CRM 選單結構與操作方式

### ⚠️ 重要：ref 每次 session 都不同

snapshot 中的 ref 在每次頁面重整後都會改變。**每次操作前必須先執行 `browser_snapshot()`**。

### 可靠的元素識別方式

**方法一：取 snapshot 後用文字比對**
```
browser_snapshot()
browser_click(element="目標選單", ref="找到的ref")
```

**方法二：browser_run_code 用文字點擊**
```javascript
browser_run_code(code="(targetText) => {\n  function tryClick(doc) {\n    const els = doc.querySelectorAll('a, td, button, li, span');\n    for (const el of els) {\n      if (el.textContent.trim() === targetText) {\n        el.click();\n        return true;\n      }\n    }\n    return false;\n  }\n  if (tryClick(document)) return 'main';\n  for (const f of document.querySelectorAll('iframe')) {\n    try {\n      if (tryClick(f.contentDocument)) return 'iframe:' + f.src;\n    } catch(e) {}\n  }\n  return 'not found';\n}")
// 用法：args=["目標選單文字"]
```

---

## 選單完整清單

### 頂部橫向分類 Tab

| 文字 | 說明 |
|------|------|
| `CRM` | 切換至 CRM 主系統 |
| `轉譯中心` | 切換至轉譯中心工作台 |

### 左側垂直圖示 Tab（CRM 內）

| 文字 | 說明 |
|------|------|
| `工作檯` | 工作檯分類（含首頁） |
| `服務管理` | 服務管理分類 |
| `客戶關係` | 客戶關係分類 |
| `系統管理` | 系統管理分類 |
| `轉譯監控` | 轉譯監控分類 |
| `電話監控` | 電話監控分類 |

### 服務管理 → 子選單

| 文字 | 說明 |
|------|------|
| `報表中心` | 服務管理報表 |
| `登入日誌` | 登入記錄 |
| `VRS服務紀錄` | VRS 服務記錄 |
| `查看 Agent Map` | Agent 地圖檢視 |
| `修改密碼` | 密碼修改 |
| `身份設定` | 身份/角色設定 |
| `通話記錄` | 電話通話記錄 |
| `聊天監控-真人客服` | 即時聊天監控 |
| `文字客服群組` | 文字客服群組管理 |
| `線上使用者` | 線上使用者列表 |
| `服務台管理` | 服務台設定 |
| `工作檯設定` | 工作檯個人化設定 |

### 系統管理 → VRS客製化功能（進階設定）

路徑：左側選單 → 基礎設定 → VRS客製化功能

| 文字 | 說明 |
|------|------|
| `服務台管理` | 服務台設定 |
| `手譯員管理` | 手譯員資料管理 |
| `滿意度調查` | 滿意度調查管理 |
| `VRS來電紀錄` | VRS 來電記錄 |
| `VRS服務統計` | VRS 服務統計報表 |
| `VRS外撥紀錄` | VRS 外撥記錄 |

---

## 中心工作台（轉譯中心）功能區

| 文字 | 位置 | 說明 |
|------|------|------|
| `電話控制` | 頂部列 | 電話控制面板 |
| `登入文字及電話客服` | 頂部右側 | 登入/登出客服狀態 |
| `就緒` | 頂部右側 | 設定狀態為「就緒」 |
| `未就緒` | 頂部右側 | 設定狀態為「未就緒」 |
| `客服工作檯` | 左側面板 | 工作台主區域 |

---

## 常用 browser_run_code 工具函式

### 全頁搜尋所有 iframe 的選單文字

```javascript
browser_run_code(code="() => {\n  const results = [];\n  const iframes = document.querySelectorAll('iframe');\n  iframes.forEach((f, i) => {\n    try {\n      const doc = f.contentDocument;\n      if (!doc) return;\n      const items = doc.querySelectorAll('a, td, li');\n      items.forEach(el => {\n        const text = el.textContent.trim();\n        if (text && text.length < 30) {\n          results.push({ iframe: i, text, href: el.href || '' });\n        }\n      });\n    } catch(e) {}\n  });\n  return results;\n}")
```

### 確認當前頁面 iframe 結構

```javascript
browser_run_code(code="() => {
  return Array.from(document.querySelectorAll('iframe')).map((f, i) => ({
    index: i,
    src: f.src.substring(0, 100),
    title: f.title || f.name || ''
  }));
}")
```

### 獲取所有 iframe 的文字內容（用於查找數據）

```javascript
browser_run_code(code="() => {
  const results = [];
  const iframes = document.querySelectorAll('iframe');
  iframes.forEach((f, i) => {
    try {
      const doc = f.contentDocument;
      if (!doc) return;
      const text = doc.body ? doc.body.innerText.substring(0, 5000) : '';
      results.push({ iframe: i, src: f.src.substring(0, 100), text: text });
    } catch(e) {
      results.push({ iframe: i, error: e.message });
    }
  });
  return results;
}")
```

---

## 常用頁面資訊

### iframe URL 對應表

| 功能 | iframe URL | 說明 |
|------|-----------|------|
| CRM 首頁/工作檯 | `.../ecp/Ecp.ChatSupervisor.page` | 包含服務群組、等候時間、座席統計 |
| 電話監控 | `.../ecp/Cti.Supervisor2.page` | 電話客服監控 |
| 首頁 | `.../ecp/Qs.Homepage.page` | 最近打開記錄 |
| VRS服務紀錄 | `.../ecp/CUS.VRSServiceLog.List.page` | VRS 服務紀錄列表 |
| 通話記錄 | `.../ecp/Ecp.CallLog.List.page` | 電話通話記錄 |
| 線上使用者 | `.../ecp/Qs.OnlineUser.List.page` | 線上使用者列表 |

### VRS服務紀錄欄位

| 欄位 | 說明 |
|------|------|
| 服務台 | 服務類型（如：手語視訊轉譯） |
| 通路 | app / web |
| 客服群組 | 文字客服 / 視訊客服 |
| 聽語障者 | 客戶姓名 |
| 手譯員 | 值機員帳號 |
| 服務開始時間 | 服務開始時間 |
| 服務結束時間 | 服務結束時間 |
| 服務項目 | 服務項目（如：購物、申請諮詢/預約、其他等） |

### 人員服務統計（座席統計）

位置：CRM 首頁 → 座席統計區塊

欄位：
- 座席姓名
- 首次登入時間
- 最後登入時間
- 最後登出時間
- 登入時長
- 未就緒時長
- 已接聽服務數
- 已服務總時長

---

## 標準操作流程範例

### 流程：登入 → 進入 CRM → 點選「VRS服務紀錄」

**第一步：登入** — 見「步驟一：登入流程」，驗證碼細節見「驗證碼辨識完整說明」。

**第二步：進入 CRM**
```
10. browser_navigate(url="https://vrs.sfaa.gov.tw/be/#/Translation/CenterDesk")
11. browser_wait_for(text="CRM", time=15)
12. browser_snapshot()
13. browser_click(element="CRM", ref="<CRM ref>")
14. browser_wait_for(text="工作檯", time=10)
```

**第三步：點選服務紀錄**
```
15. browser_snapshot()
16. browser_click(element="服務管理", ref="<服務管理 ref>")
17. browser_snapshot()
18. browser_click(element="VRS服務紀錄", ref="<VRS服務紀錄 ref>")
19. browser_take_screenshot()
```

---

## 故障排除

| 問題 | 原因 | 解法 |
|------|------|------|
| 驗證碼辨識錯誤，登入失敗 | 圖形不清楚 | 點擊「換一張圖」重新截圖，然後使用 Claude 模型重新辨識 |
| 頁面空白，只有橘色 header | 中心工作台 ECP iframe 載入中 | 等待 15-20 秒後再 browser_snapshot |
| 點擊後無反應 | ref 已過期（頁面重整） | 重新 browser_snapshot 取得新 ref |
| 通知彈窗無法關閉 | Chrome 原生 UI | 用 browser_press_key(key="Escape") 或 Tab+Tab+Enter |
| iframe 內容無法存取 | 跨域限制 | 使用 browser_run_code 在 iframe 內執行 |
| 登入後跳回登入頁 | Session 過期或 token 失效 | 重新執行完整登入流程 |
| 找不到選單項目 | 選單分類未展開 | 先點父層分類，再 browser_snapshot 找子項 |

---

## 驗證碼辨識完整說明

**特性：** 6 位純數字，圖片有扭曲、干擾線 → 一般 OCR/Tesseract 無法處理，`task()`（MiniMax）也無圖片辨識能力。**唯一可行方式：Claude 直接從截圖視覺辨識**（多模態能力，不需 opencode run/OCR）。

> ⚠️ **關鍵規則：驗證碼與 session 綁定**，截圖→辨識→填入→送出必須連續完成，中途點「換一張圖」會讓已辨識的結果立即失效，必須整組重來。

**正確順序：**
```
1. browser_take_screenshot(element="驗證碼圖片", ref="<ref>", filename="captcha.png")
2. # Claude 直接從截圖結果辨識 6 位數字
3. browser_type(element="驗證碼", ref="<驗證碼ref>", text="<辨識到的驗證碼>")
4. browser_click(element="登入按鈕", ref="<登入ref>")   # 不要中途點「換一張圖」！
```

**失敗處理**（顯示「驗證碼不符」時）：點「換一張圖」→ 回到步驟 1 整組重來（最多 3 次）。

---

## ECP API 直接存取（高效率批次查詢）

> 適合大量資料擷取，不需 Playwright 操作介面，速度遠快於瀏覽器自動化。

### 取得 JSESSIONID

登入後從瀏覽器取得 Session Cookie：

```javascript
// browser_run_code 取得 cookie
browser_run_code(code="() => { return document.cookie; }")
// 或透過 Playwright network requests 攔截 API 回應中的 Set-Cookie
```

Cookie 格式：`JSESSIONID=<32位hex>; Language=zh-tw`

> ⚠️ Session 有時效性，若 API 回傳 401/403 或空資料，需重新登入取得新 JSESSIONID。

---

### VRS 服務紀錄批次 API

**端點：**
```
POST https://sfaa-vrsp-ecp.qbicloud.com/ecp/qsvd-list/CUS.VRSServiceLog.getListData.data
Content-Type: application/json
Cookie: JSESSIONID=<token>; Language=zh-tw
```

**請求 Payload：**
```json
{
  "listId": "18a934a2-9440-0fb0-5a35-005056b80556",
  "keyword": "",
  "queryFormRecent": {},
  "pageIndex": 1,
  "conditions": [
    {"fieldName": "U_ServiceStartTime", "operator": "GreatEqual", "value": "2025-01-01 00:00:00"},
    {"fieldName": "U_ServiceStartTime", "operator": "LessEqual", "value": "2025-12-31 23:59:59"}
  ]
}
```

**回應結構：**
```json
{
  "data": [ ...records... ],
  "hasNextPage": true,
  "totalCount": 6942
}
```

**欄位對照：**

| 欄位名稱 | 說明 |
|---------|------|
| `FId` | 紀錄 UUID |
| `U_ContactId2` | 聽語障者姓名 |
| `U_Channel` | 通路（app / web） |
| `U_WorkGroupName` | 客服群組 |
| `U_Professional` | 服務項目 |
| `U_ServiceDeskName` | 服務台 |
| `U_ServiceStartTime` | 服務開始時間 |
| `U_ServiceCloseTime` | 服務結束時間 |
| `U_AgentName` | 手譯員帳號 |

**Python 批次抓取範例：**
```python
import requests, json

JSESSIONID = "填入實際值"
headers = {
    "Content-Type": "application/json",
    "Cookie": f"JSESSIONID={JSESSIONID}; Language=zh-tw"
}

def fetch_all_records(start_date, end_date):
    url = "https://sfaa-vrsp-ecp.qbicloud.com/ecp/qsvd-list/CUS.VRSServiceLog.getListData.data"
    all_records = []
    page = 1
    while True:
        payload = {
            "listId": "18a934a2-9440-0fb0-5a35-005056b80556",
            "keyword": "",
            "queryFormRecent": {},
            "pageIndex": page,
            "conditions": [
                {"fieldName": "U_ServiceStartTime", "operator": "GreatEqual", "value": f"{start_date} 00:00:00"},
                {"fieldName": "U_ServiceStartTime", "operator": "LessEqual", "value": f"{end_date} 23:59:59"}
            ]
        }
        resp = requests.post(url, headers=headers, json=payload)
        data = resp.json()
        records = data.get("data", [])
        all_records.extend(records)
        print(f"  Page {page}: {len(records)} records (total: {len(all_records)})")
        if not data.get("hasNextPage", False):
            break
        page += 1
    return all_records

# 2025 年
records_2025 = fetch_all_records("2025-01-01", "2025-12-31")
# 2026 年
records_2026 = fetch_all_records("2026-01-01", "2026-12-31")
```

**匯出 Excel（openpyxl）：**
```python
import openpyxl
from openpyxl.styles import Font, PatternFill, Alignment

def export_to_excel(all_records, records_2025, records_2026, filepath):
    wb = openpyxl.Workbook()
    columns = [
        ("FId","紀錄ID"), ("U_ContactId2","聽語障者"),
        ("U_Channel","通路"), ("U_WorkGroupName","客服群組"),
        ("U_Professional","服務項目"), ("U_ServiceDeskName","服務台"),
        ("U_ServiceStartTime","服務開始時間"), ("U_AgentName","手譯員"),
        ("U_ServiceCloseTime","服務結束時間")
    ]
    for sheet_name, records in [("全部紀錄", all_records), ("2025年", records_2025), ("2026年", records_2026)]:
        ws = wb.active if sheet_name == "全部紀錄" else wb.create_sheet(sheet_name)
        ws.title = sheet_name
        ws.append([ch for _, ch in columns])
        for rec in records:
            ws.append([rec.get(f, "") for f, _ in columns])
    wb.save(filepath)
```

---

### TA102 座席服務紀錄表 API（已驗證）

**✅ 確認可用端點（GET，直接下載 XLS）：**

```
GET https://sfaa-vrsp-ecp.qbicloud.com/ecp/qsvd-report/frameset
  ?__report=ChatReport/TA102.rptdesign
  &pStartTime=2026-01-01+00:00
  &pEndTime=2026-03-26+23:59
  &pAgentId=<座席UUID>
  &language=zh-tw
  &__parameterpage=false
  &isDesign=false
  &__format=xls
Cookie: JSESSIONID=<token>; Language=zh-tw
```

> ⚠️ `__format=csv` 不支援（BIRT 伺服器錯誤），只能用 `xls`（SpreadsheetML XML 格式，Excel 可直接開啟）

**TA102 欄位（已確認）：**

| 欄位 | 說明 |
|------|------|
| 開始時間 | ISO format: `2026-01-02T11:11:43.000` |
| 結束時間 | ISO format |
| 對方號碼 | 聽語障者姓名（如 `(徐麗霞)`） |
| 群組 | 如 `1002(視訊客服)` |
| 佇列持續時間 | 振鈴等待時長 |
| 通話持續時間 | 實際通話時長 |
| 是否SLT | 是 / 否 |
| 登出狀態 | 如 `使用者掛斷` |
| 通話識別碼 | UUID |

**必要參數 `pAgentId`：** 座席 UUID（非帳號名稱）

| 座席帳號 | UUID |
|---------|------|
| CS0009 | `19006462-08b0-0363-34fe-a2ea64699c5e` |
| 其他座席 | 從報表中心 Entity Picker 操作時攔截網路請求取得 |

**Python 批次下載並解析 TA102：**
```python
import requests, re

JSESSIONID = "填入實際值"
headers = {"Cookie": f"JSESSIONID={JSESSIONID}; Language=zh-tw"}

def fetch_ta102_xls(agent_uuid, start_time, end_time, save_path):
    """下載單一座席的 TA102 報表（SpreadsheetML XLS）"""
    url = "https://sfaa-vrsp-ecp.qbicloud.com/ecp/qsvd-report/frameset"
    params = {
        "__report": "ChatReport/TA102.rptdesign",
        "pStartTime": start_time,  # e.g. "2026-01-01 00:00"
        "pEndTime": end_time,      # e.g. "2026-03-26 23:59"
        "pAgentId": agent_uuid,
        "language": "zh-tw",
        "__parameterpage": "false",
        "isDesign": "false",
        "__format": "xls"
    }
    resp = requests.get(url, headers=headers, params=params)
    with open(save_path, 'wb') as f:
        f.write(resp.content)
    return resp.status_code

def parse_ta102_xls(filepath):
    """解析 SpreadsheetML XLS，回傳資料列清單"""
    with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
        content = f.read()
    rows = re.findall(r'<Row[^>]*>(.*?)</Row>', content, re.DOTALL)
    data = []
    for row in rows:
        cells = re.findall(r'<Data[^>]*>(.*?)</Data>', row)
        if cells:
            # 清除 HTML entities
            cells = [re.sub(r'&#\d+;', ' ', c).strip() for c in cells]
            data.append(cells)
    return data

# 使用範例
fetch_ta102_xls("19006462-08b0-0363-34fe-a2ea64699c5e",
                "2026-01-01 00:00", "2026-03-26 23:59",
                "C:/Users/HCH/TA102_CS0009.xls")
rows = parse_ta102_xls("C:/Users/HCH/TA102_CS0009.xls")
print(f"Parsed {len(rows)} rows")  # 包含標題行
```

**其他報表格式（同樣支援）：**

| `__format` | 大小 | 說明 |
|-----------|------|------|
| `xls` | ~168KB | SpreadsheetML（推薦，中文正確） |
| `pdf` | ~30KB | PDF 格式 |
| `doc` | ~2MB | Word 文件 |
| `csv` | ❌ | 不支援 |

**❌ 失效端點：**
- `POST /ecp/Cti.Report.TA102.data` → payload 格式錯誤（begin 0, end -1）
- `POST /ecp/report/TA102.data` → 單元編碼不存在

---

## 報表中心（客服中心運營報表）

### 進入報表中心

路徑：服務管理 → 報表中心

```
browser_navigate(url="https://sfaa-vrsp-ecp.qbicloud.com/ecp/Qs.Report.Center.page")
```

### 選單結構

| 層級 | 選單 |
|------|------|
| 一級 | 客服中心電話營運報表 |
| 一級 | 客服中心運營報表 |
| 二級（座席報表） | 詳細報表 / 統計報表 |
| 三級（詳細報表） | TA101-座席狀態明細表 / TA102-座席服務紀錄表 / TA103-座席平均回覆處理時間明細表 / TA104-座席重新佇列明細表 |

### TA101 座席狀態明細表

這是最常用的座席狀態報表。

#### 查詢流程

1. 依序點選：客服中心運營報表 → 座席報表 → 詳細報表 → TA101
2. 填寫查詢表單：
   - **開始時間**：`YYYY-MM-DD 00:00`（例如 2026-03-01 00:00）
   - **結束時間**：`YYYY-MM-DD 23:59`（例如 2026-03-26 23:59）
   - **座席**：需使用 JuiMultiEntityBox 放大鏡挑選器

#### 座席選擇方式（重要）

**不能直接輸入**，必須用挑選器：
1. 點擊座席欄位的 **放大鏡按鈕** (`.JuiMultiEntityBoxMagnifier`)
2. 彈出選擇對話框後，在左側清單中**雙擊**目標座席
3. 座席會移動到右側已選區
4. 點擊 **確定** 按鈕確認

```javascript
// 用 JavaScript 點擊放大鏡按鈕
browser_run_code(code="() => {
  const iframe = document.querySelector('#Report');
  const iframeDoc = iframe.contentDocument || iframe.contentWindow.document;
  const magnifier = iframeDoc.querySelector('.JuiMultiEntityBoxMagnifier');
  if (magnifier) { magnifier.click(); return 'Clicked magnifier'; }
  return 'Magnifier not found';
}")
```

#### 執行查詢

1. 點擊 **執行** 按鈕
2. 等待 BIRT Report Viewer 載入（約 3-5 秒）
3. 確認有「Showing page X of Y」表示成功

#### 欄位說明

| 欄位 | 說明 |
|------|------|
| 時間 | 狀態變更時間 |
| 座席狀態 | 登入 / 登出 / 就緒 / 未就緒 |
| 未就緒原因 | 空白（就緒/登入/登出時）、休息 / 會議 / 電話進線未就緒 / 建檔中 等 |

### 匯出 CSV（Export data）

BIRT Report Viewer 支援匯出 CSV 格式，**不支援直接匯出 Excel**。

#### 匯出步驟

1. 點擊 **Export data** 按鈕
2. 在對話框中：
   - **Available result sets**：選擇 `table1`（不是 ELEMENT_107）
   - 點擊 **Add all** 加入所有欄位
   - **Export format**：CSV(*.csv)（預設）
   - **Output encoding**：UTF-8（預設）
3. 點擊 **OK**

#### 欄位對照

table1 可用欄位：
| 欄位 | 說明 |
|------|------|
| FSeatId | 座席ID（如 5109） |
| FUnreadyReasonCode | 未就緒原因代碼 |
| FStatus | 狀態（登入/登出/就緒/未就緒） |
| FSeatName | 座席名稱（如 CS0009） |
| FSeatIdN | 完整名稱（如 5109(CS0009)） |
| FCreateTime | 變更時間 |

### API 端點（僅供參考，無法直接外部呼叫）

**這是 BIRT 報表系統，不是 REST API，需要登入 Session 才能呼叫。**

```
POST https://sfaa-vrsp-ecp.qbicloud.com/ecp/qsvd-report/frameset
```

**請求參數：**
| 參數 | 說明 | 範例值 |
|------|------|--------|
| `__report` | 報表設計檔 | `ChatReport/TA101.rptdesign` |
| `pStartTime` | 開始時間 | `2026-03-01 00:00` |
| `pEndTime` | 結束時間 | `2026-03-26 23:59` |
| `pAgentId` | 座席UUID | `19006462-08b0-0363-34fe-a2ea64699c5e`（CS0009） |
| `language` | 語言 | `zh-tw` |
| `__parameterpage` | 不顯示參數頁 | `false` |
| `isDesign` | 非設計模式 | `false` |
| `__sessionId` | 時間戳記 | `YYYYMMDD_HHMMSS_MMM` |

### 座席存在性確認

**CS0006~CS0030 範圍內，存在的座席（17個）：**

| 座席 | 狀態 |
|------|------|
| CS0007 | 存在 |
| CS0008 | 存在 |
| CS0009 | 存在 |
| CS0011 | 存在 |
| CS0012 | 存在 |
| CS0013 | 存在 |
| CS0014 | 存在 |
| CS0015 | 存在 |
| CS0017 | 存在 |
| CS0018 | 存在 |
| CS0021 | 存在 |
| CS0022 | 存在 |
| CS0023 | 存在 |
| CS0024 | 存在 |
| CS0025 | 存在 |
| CS0026 | 存在（需搜尋） |
| CS0030 | 存在（需搜尋） |

**不存在的座席（8個）：** CS0006, CS0010, CS0016, CS0019, CS0020, CS0027, CS0028, CS0029

> ⚠️ 這是 2026-03-26 的快照，座席可能動態新增或刪除。

### Entity Picker 搜尋技巧

**重要發現**：Entity picker 對話框首頁只顯示部分座席（約 20 個），**不會列出全部**。需要用搜尋功能找特定座席。

**操作方式：**
1. 點擊放大鏡開啟 Entity picker
2. 在「部門/姓名」文字框輸入座席名稱或帳號
3. 按 Enter 搜尋
4. 雙擊結果選擇

**適用情境：**
- CS0026、CS0030 等不在首頁列表的座席
- 大量座席查詢時，可先搜尋確認存在性

### Agent ID 對照表（完整）

| 顯示名稱 | 內部 UUID |
|---------|----------|
| CS0007 | （請從系統查詢） |
| CS0008 | （請從系統查詢） |
| CS0009 | `19006462-08b0-0363-34fe-a2ea64699c5e` |
| CS0011~CS0030 | （請從系統查詢） |

### CSV 匯出重要發現

**BIRT 編碼問題（重要）：**

伺服器端在匯出 CSV 時有編碼 bug，中文字元會損壞：
- 原始正確：登入、登出、就緒、未就緒、休息、會議 等
- 匯出後損壞：變成 ?鶴?、?芸停蝺? 等亂碼

**目前狀態：**
- CSV 檔案編碼：聲稱是 UTF-8，實際內容也確實是 UTF-8 位元組
- 但 UTF-8 解碼出來的中文是錯誤的（鶴 而非 登入）
- 使用 MS950/Big5 解碼也同樣失敗
- **結論**：這是 BIRT 伺服器在產生 CSV 時就已經將中文字元錯誤編碼，無法透過重新編碼修復

**建議：**
- 如需正確的中文資料，改用「Export report」（匯出報表格式）而非「Export data」（匯出原始數據）
- 或者直接從網頁介面複製貼上（顯示正常的中文）

### 常見問題

| 問題 | 解法 |
|------|------|
| Export data 顯示 "Session timeout" | 點擊 "Run report" 重新產生文件，再 Export |
| 匯出的 CSV 只有 2 行 | 選擇了 ELEMENT_107（metadata），需改選 table1 |
| Session timeout 或文件不存在 | 點擊 Run report 重新執行查詢 |
| CSV 中文顯示亂碼 | 這是 BIRT 伺服器端 bug，無法透過編碼轉換修復，請使用 Export report 替代 |
| 找不到特定座席（如 CS0026） | 使用 Entity picker 的搜尋功能，輸入帳號後按 Enter 搜尋 |

---

## 獨立工具腳本（Standalone Python Scripts）

> 以下腳本位於技能目錄，可獨立執行，不依賴 Playwright MCP 工具。
> 適合定時排程（如 Windows 工作排程器）或直接雙擊執行。

### 環境需求

```bash
pip install playwright openpyxl Pillow requests
playwright install chromium
```

---

### 合併版腳本：scripts/vrs_online_check.py（推薦使用）

**檔案：** `scripts/vrs_online_check.py`

**邏輯：**
1. **優先**：使用本地 Ollama（qwen3-vl:8b）自動辨識驗證碼
2. **Fallback**：若 Ollama 連不上（主機未開機、模型不存在等），自動改為手動輸入驗證碼

**使用方式：**
```bash
cd C:\Users\HCH\.claude\skills\vrs
python scripts/vrs_online_check.py
```

**Ollama 需求（可選）：**
- Ollama 運行於 `http://10.145.119.234:11434`
- 模型：`qwen3-vl:8b`（需預先拉取：`ollama pull qwen3-vl:8b`）
- 若 Ollama 未運行，腳本會偵測到並提示手動輸入，不會失敗

**輸出範例（Ollama 可用）：**
```
[Ollama] 辨識: 483921（原始回應: 483921）
已點擊「確定」處理重複登入提醒
登入成功
截圖已存至 vrs_status.png

=== VRS 就緒/未就緒人數 ===

【文字客服】
  就緒: 3（空閒 1 + 忙線 2）
  未就緒: 5

【視訊客服】
  就緒: 2（空閒 1 + 忙線 1）
  未就緒: 3

【等候情況】
  線上等待總人數: 0
  最久等候時間: 0 秒
```

**輸出範例（Ollama 不可用）：**
```
[Ollama] 連線失敗: [Errno 11001] getaddrinfo failed，將改用手動輸入
驗證碼截圖已存至 vrs_captcha.png
請在瀏覽器中輸入驗證碼，輸入完成後按 Enter 繼續:
```

---

### 舊版腳本（仍保留，僅供參考）

| 檔案 | 說明 |
|------|------|
| `scripts/查詢就緒人數人數.py` | Ollama 版（獨立腳本） |
| `scripts/線上等待總人數.py` | MiniMax API 版（獨立腳本） |

> 合併版已整合兩者功能，建議使用 `scripts/vrs_online_check.py`

---

### 通用函式：parse_online_stats_text

兩個腳本共用同一個解析邏輯，可用於其他腳本：

```python
import re

def parse_online_stats_text(snapshot: str) -> dict:
    stats = {"文字客服": None, "視訊客服": None, "等候人數": 0, "最久等候": "0"}

    for line in snapshot.split("\n"):
        # 文字客服解析：等候 最長等待 總成員 登出 登入 未就緒 已就緒-空閒 已就緒-忙線
        m = re.search(
            r"文字客服\s+(\d+)\s+([\d:]+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)",
            line
        )
        if m:
            stats["文字客服"] = {
                "等候": m.group(1), "最長等待": m.group(2),
                "總成員": m.group(3), "登出": m.group(4),
                "登入": m.group(5), "未就緒": m.group(6),
                "已就緒-空閒": m.group(7), "已就緒-忙線": m.group(8),
            }

        # 視訊客服解析（格式相同）
        m = re.search(
            r"視訊客服\s+(\d+)\s+([\d:]+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)",
            line
        )
        if m:
            stats["視訊客服"] = {
                "等候": m.group(1), "最長等待": m.group(2),
                "總成員": m.group(3), "登出": m.group(4),
                "登入": m.group(5), "未就緒": m.group(6),
                "已就緒-空閒": m.group(7), "已就緒-忙線": m.group(8),
            }

        # 等候人數
        m = re.search(r"線上等待總人數:(\d+)", line)
        if m:
            stats["等候人數"] = int(m.group(1))

        # 最久等候
        m = re.search(r"線上等待最久時間:(\d+)", line)
        if m:
            stats["最久等候"] = m.group(1)

    return stats
```

**解析結果結構：**
```python
{
    "文字客服": {
        "等候": "0", "最長等待": "00:00", "總成員": "8",
        "登出": "3", "登入": "8", "未就緒": "5",
        "已就緒-空閒": "1", "已就緒-忙線": "2"
    },
    "視訊客服": { ... },
    "等候人數": 0,
    "最久等候": "0"
}
```

---

## Conformance Addendum

## When to Use
Use when operating the 手語視訊轉譯中心後台系統 (VRS) - including login, navigating to CRM or 中心工作台, clicking menu items, handling browser permission popups, or automating any workflow within sfaa-vrsp-ecp.qbicloud.com or vrs.sfaa.gov.tw

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
