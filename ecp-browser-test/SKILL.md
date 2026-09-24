---
name: ecp-browser-test
description: 用 Playwright MCP 測試 ECP (aipower) 所有功能選單。點選所有葉節點、截圖驗證、回報錯誤。用在：冒煙測試、部署後驗證、新功能探索。
triggers:
  - ecp test
  - ecp 測試
  - aipower test
  - 測試功能
  - 點選選單
argument-hint: "[smoke|full|section <name>]"
---

# ECP Browser Test Skill

## 前置條件

1. Chrome 已用 debug port 9222 啟動（執行桌面的 `啟動Chrome-Debug.ps1`）
2. Playwright MCP 已連線（`mcp__plugin_playwright_playwright__*` 工具可用）
3. ECP 已登入，或需要登入時帳號 `administrator`（密碼見 `ecp-pwd` skill）

## 連線與導航

```javascript
// 確認已連到 Chrome
await browser_navigate({ url: 'https://hch.james-huang.org/aipower/Qs.MainFrame.page' })
// 若出現登入頁，見下方「登入流程」
```

### 登入流程

```javascript
// 填帳號密碼
await browser_fill_form({ fields: [
  { selector: '#FAccount', value: 'administrator' },
  { selector: '#FPassword', value: '<見 ecp-pwd skill>' }
]})
// 點登入
// 若出現「已登入，是否繼續？」→ 點確定
await browser_handle_dialog({ accept: true })
```

## 核心 DOM 知識

### 選單結構

```
#MenuZone
  .JuiOutlookBarItemHead   ← 群組標題（可展開/收合），有 onclick
  .JuiTreeTextCell.JuiTreeLeafIcon  ← 葉節點（實際功能項），有 onclick
```

### 列出所有葉節點（JS）

```javascript
() => {
  const items = document.querySelectorAll('#MenuZone .JuiTreeTextCell.JuiTreeLeafIcon');
  return Array.from(items).map((el, i) => `[${i}] ${el.innerText?.trim()}`);
}
```

### 展開群組（JS）

```javascript
() => {
  const headers = document.querySelectorAll('#MenuZone .JuiTitle.JuiOutlookBarItemHead');
  for (const h of headers) {
    if (h.innerText?.includes('TARGET_NAME')) { h.click(); return 'ok'; }
  }
}
```

### 批次點選指定範圍（JS）

```javascript
() => {
  const items = document.querySelectorAll('#MenuZone .JuiTreeTextCell.JuiTreeLeafIcon');
  const clicked = [];
  Array.from(items).slice(START_INDEX, END_INDEX).forEach(el => {
    clicked.push(el.innerText?.trim());
    el.click();
  });
  return clicked;
}
```

### 關閉 ECP 對話框

ECP 對話框**用「返回」關閉**，不是 × 或 Escape：

```javascript
() => {
  // 方法一：找錯誤 iframe 的父層 JuiDialogClose
  const iframes = document.querySelectorAll('iframe');
  for (const f of iframes) {
    try {
      if (!f.contentDocument?.body?.innerText?.includes('ERROR_TEXT')) continue;
      let el = f.parentElement;
      for (let i = 0; i < 10; i++) {
        if (!el) break;
        const btn = el.querySelector('[class*="JuiDialogClose"],[class*="JuiDialogButton"]');
        if (btn) { btn.click(); return 'closed'; }
        el = el.parentElement;
      }
    } catch(e) {}
  }
}
```

## 完整選單索引（2026-06-23 版本）

### 工作檯（TabZone 左側欄）

| 索引 | 名稱 |
|------|------|
| 0 | 通話記錄 |
| 1 | 文字客服 |
| 2 | AI Proxy 設定 |
| 3 | CBM-Lite 設定 |
| 4 | LIFF 訂單總表 |
| 5 | AI Proxy 設定（未完成）|
| 6 | 聯絡人 |

### 系統管理 → 基礎設定

| 索引 | 名稱 |
|------|------|
| 7 | 線上使用者 |
| 8 | 單元設定 |
| 9 | 頁面設定 |
| 10 | 資料表匯出 |
| 11 | 功能表設定 |
| 12 | hch（工作台設定）|
| 13 | 工作檯設定 |
| 14 | 部門 |
| 15 | 職務 |
| 16 | 員工 |
| 17 | 角色 |
| 18 | 團隊 |
| 19 | 社團 |
| 20 | 服務號 |
| 21 | 帳號 |
| 22 | 身份類型 |
| 23 | 線上使用者 |
| 24 | 登入日誌 |
| 25 | 業務日誌 |
| 26 | 事件日誌 |
| 27 | SQL 升級日誌 |
| 28 | SQL 執行日誌 |
| 29 | 性能監控 |
| 30 | 快取監控 |
| 31 | 附件閱覽轉換監控 |
| 32 | 系統參數 |
| 33 | 公司參數 |
| 34 | 參數定義 |
| 35 | 系統首頁設定 |
| 36 | 圖形首頁設定 |
| 37 | 計時器 |
| 38 | 計時器日誌 |
| 39 | 郵件範本 |
| 40 | 簡訊範本 |
| 41 | 流程管理 |
| 42 | 工作項管理 |
| 43 | 字典設定 |
| 44 | 通知設定 |
| 45 | 圖片 |
| 46 | 幣種設定 |
| 47 | 簡訊日誌 |
| 48 | 系統郵件日誌 |
| 49 | 資料連結 |
| 50 | Ecp圖片 |
| 51 | 數據同步設定 |
| 52 | 點數活動設定 |
| 53 | AP頁籤 |

### 系統管理 → 進階設定

| 索引 | 群組 | 名稱 |
|------|------|------|
| 54 | 單元 | 單元設定 |
| 55 | 單元 | 編號設定 |
| 56 | 單元 | 單元轉換設定 |
| 57 | 單元 | 單元討論設定 |
| 58 | 頁面 | 頁面設定 |
| 59 | 頁面 | 頁面集設定 |
| 60 | 頁面 | 特殊路徑設定 |
| 61 | 頁面 | 通用指令碼設定 |
| 62 | 統計分析 | 圖表設定 |
| 63 | 統計分析 | 報表設定 |
| 64 | 統計分析 | 單據設定 |
| 65 | 伺服器 | 應用管理 |
| 66 | 伺服器 | 集群管理 |
| 67 | 資料庫 | 索引管理 |
| 68 | 資料庫 | 執行 SQL |
| 69 | 資料庫 | 資料表匯出 |
| 70 | 資料庫 | 資料庫標記 |
| 71 | 移動端設定 | 手機功能表 |
| 72 | 資料整合 | Token 設定 |
| 73 | 資料整合 | 遠端 API 組 |
| 74 | 資料整合 | 遠端 API |
| 75 | 資料整合 | 外接 WebSocket |
| 76 | 資料整合 | 跨域資源分享 |
| 77 | 資料整合 | 直連API組 |
| 78 | 其它 | 語言設定 |
| 79 | 其它 | 功能表設定 |
| 80 | 其它 | 圖標功能表 |
| 81 | 其它 | 許可權設定 |
| 82 | 其它 | 關聯許可權 |
| 83 | 其它 | 從屬頁面許可權 |
| 84 | 其它 | 文字資源 |
| 85 | 其它 | 工作流定義 |
| 86 | 其它 | 本地 API 組 |
| 87 | 其它 | 本地 API |
| 88 | 其它 | WebService |
| 89 | 其它 | Aile通知設定 |
| 90 | 其它 | Aile通知日誌 |

### 系統管理 → 開發環境項目

| 索引 | 名稱 | 備注 |
|------|------|------|
| 91 | AIFF應用管理 | ⚠️ 已知 bug：FServiceConsultSchemeId 欄位不存在 |
| 92 | 外部應用 | 正常 |

### 各類報表

- 每日報表、每月報表、年度報表（子項依部署而異）

## 測試流程

### 全選單 iframe 掃描（已腳本化）

當使用者要求「全部翻一下」並統計 iframe，不要重新手動推導流程，直接跑：

```powershell
node C:\Users\HCH\.claude\skills\ecp-browser-test\scripts/scan-aipower-iframes.js 9222 450
```

參數：第一個是 Chrome CDP port，第二個是每個選單項目點擊後等待毫秒數。腳本會附著既有 Chrome，不會啟動新 Chrome，會逐一點擊 `#MenuZone .JuiTreeTextCell.JuiTreeLeafIcon`，最後輸出 `iframeCount`、`tabIframeCount`、`secondaryHomepageCount`、`visibleIframeCount` 和 iframe 清單。

> ⚠️ **2026-07-24 更新**：舊版範例寫死一個已不存在的舊 port（舊版 `dev-aipower` skill 接的
> `C:\com\chainsea` 專用 CDP），那套安裝和對應的 MCP server 已不存在。第一個參數要改成當下
> 實際接管的 Chrome CDP port（目前共用的 `playwright-vrs` 固定接 **9222**，見 `connect-chrome` skill；
> 若為其他 aipower 實例建立了專用 CDP，改成那個實例的 port）。

### 冒煙測試（smoke）

點選各 tab 群組的第一個功能，確認頁面能載入不報錯：

```javascript
// 點基礎設定第一項
await browser_evaluate({ function: '() => { document.querySelectorAll("#MenuZone .JuiTreeTextCell.JuiTreeLeafIcon")[7].click() }' })
await browser_take_screenshot({ filename: 'smoke-基礎設定.png' })
// 重複各 section 首項
```

### 全量測試（full）

1. 確認四個 tab（工作檯、系統管理、各類報表）都切換過
2. 在系統管理下依序展開「基礎設定」→「進階設定」→「開發環境項目」
3. 用批次點選 JS 逐段點完（slice 範圍見索引表）
4. 每段結束截圖記錄頁籤狀態
5. 遇到錯誤對話框 → 找 `JuiDialogClose` 關閉，記錄錯誤內容

### 特定功能測試（section）

```javascript
// 例：只測進階設定
() => {
  const items = document.querySelectorAll('#MenuZone .JuiTreeTextCell.JuiTreeLeafIcon');
  return Array.from(items).slice(54, 91).forEach(el => el.click());
}
```

## 回報格式

測試結束輸出：

```
## ECP 功能測試報告 <日期>

### 通過（無錯誤）
- [index] 功能名 ✓

### 錯誤
- [index] 功能名 ✗ — 錯誤訊息

### 已知問題（略過）
- [91] AIFF應用管理 — FServiceConsultSchemeId 欄位不存在（已知）
```

## 坑

1. **批次點選後頁籤可能撐爆**：ECP 每次點選都會開新 tab，全量點完可能有 90+ 個 tab。如需避免，逐項點 → 截圖 → 關 tab。
2. **群組未展開**：`進階設定`、`開發環境項目` 預設收合，要先 `.click()` header 才能看到葉節點。
3. **索引會偏移**：若其他 tab（工作檯/報表）的葉節點數量改變，系統管理的索引也會跟著偏移。建議先執行「列出所有葉節點」確認索引，或用文字比對取代 slice。
4. **ECP 關閉對話框**：一律點 `JuiDialogClose`（返回），不要按 Escape 或找 ×。

---

## Conformance Addendum

## When to Use
用 Playwright MCP 測試 ECP (aipower) 所有功能選單。點選所有葉節點、截圖驗證、回報錯誤。用在：冒煙測試、部署後驗證、新功能探索。

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
