---
name: ecp
version: 1.0.0
description: ECP (eContact Platform) system knowledge base. Use when working with ECP APIs, UI automation, architecture, workflows, chat, external channel integrations, or employee account management on the Quicksilver Framework.
description_zh: ECP（eContact Platform）系統知識庫。用於 ECP API 呼叫、UI 自動化、系統架構、工作流引擎、即時通訊、外部管道整合或員工帳號管理。
---

## Skill: ecp

File references (@path) in this skill are relative to the directory where SKILL.md is located.

---

## 📎 附件參考

### 功能表結構對照表

修正後的 ECP 功能表結構 CSV 檔案：
- **檔案**: `@path(qbicrm-complete-v4-corrected.csv)`
- **說明**: 包含 ECP 系統完整功能表結構（424項），已根據實際系統校正路徑

### API 完整文檔 (2026-04-01)

ECP 系統 API 完整呼叫參考（100+ 端點，已實際驗證）：
- **檔案**: `@path(ECP-API-2026-04-01.md)`
- **說明**: 包含所有 Qs.*/Ecp.*/Wf.* 模組的 CRUD API 格式、分頁語法、常見錯誤碼

### 🆕 系統架構完整分析 (2026-04-01)

**深度架構分析** - 涵蓋前後端完整架構、Quicksilver Framework、數據庫 Schema：
- **檔案**: `@path(ECP-ARCHITECTURE-2026-04-01.md)`
- **說明**: 
  - 前端 Utility.invoke() / syncInvoke() API 呼叫機制
  - 後端 Action/Service/Dao 分層架構
  - API 路由：`Module.Entity.method.data` → Java 反射呼叫
  - 核心數據表：TsAccount, TsUser, TsIdentityType, TsAccountIdentity
  - ECP 業務表：TcContact, TcChatRoom, TcChatMessage, TcChatWorkGroup
  - 表關係圖：帳號 ↔ 身份 ↔ 用戶 ↔ 業務資料

#### 12項深度研究（2026-04-01 完成）

| # | 主題 | 內容摘要 |
|---|------|---------|
| 1 | **Java ActionImpl 實作** | 4層繼承結構：`BaseActionImpl` → `EntityActionImpl` → `BusinessEntityActionImpl` → `*ActionImpl`。Core CRUD 在 `EntityActionImpl`（72KB），自定義方法如 `merge()`, `hasAuth()`, `checkPwd()` 在具體實作中 |
| 2 | **Chat WebSocket 即時訊息** | ECP 使用自訂 **JSocket** 函式庫（`jocket.js` 1048行）。雙模式：Internal Chat 用 `Jocket` 類，External WebChat 用原生 `WebSocket`。訊息流程：WebSocket → jocket handlePacket → MainFrame.doJocketMessage → type-specific handler |
| 3 | **Workflow 工作流引擎** | 定義表：`TwWorkflow`, `TwNode`（Start/End/Manual/Auto）, `TwLine`。執行表：`TwProcess`, `TwWorkItem`（waiting/completed）。關鍵 API：`Wf.WorkItem.finish/draw/transfer/clone` |
| 4 | **外部管道整合** | 支援 Facebook/LINE/Instagram/Telegram/WebChat/SMS。核心類：`ChannelHome`, `OfflineApi`, `OfflineGwApi`。訊息流：外部管道 → OfflineGwApi → OfflineReplyService → TcOfflineRoom |

---

## 🆕 ECP 清單頁面架構與 UI 刪除流程（2026-04-01 實測）

### 清單頁面 4-Table 同步滾動架構

ECP 所有清單頁面（如 `Ecp.Customer.List.page`）使用 **4 個同步表格** 實現列凍結（row number）和數據區的聯動：

| 表格 class | 位置 | 寬度 | 說明 |
|------------|------|------|------|
| `JuiListTable JuiListHeadTable` | `left: 0` | 80px | 表頭行號列（Frozen） |
| `JuiListTable JuiListHeadTable` | `left: -261` | 1041px | 主表頭（與數據列同步滾動） |
| `JuiListTable JuiListLeftTable` | `top: 52, left: 0` | 80px | 行號列（Frozen，row height=25px） |
| `JuiListTable JuiListDataTable` | `top: 52, left: -261` | 1041px | 數據區（與主表頭同步滾動） |

**特點**：
- 數據行的 x 座標幾乎都是負數（`left: -261`），因為被左側 frozen 列推出視窗外
- 點擊行號列的 **第二個 cell**（index=1，class=`JuiListCheckCell`）來選中行
- 選中後 row 追加 class `JuiListSelectedRow`
- **沒有** checkbox 全選列 — 只能單選，無法批量選取

### 行選中流程（重要！）

```javascript
// ✅ 正確方式：JS 點擊 check cell（繞過 viewport 座標問題）
// 使用 eval 執行 JS
playwright-cli eval "() => {
  const leftTable = document.querySelector('.JuiListLeftTable');
  const firstRow = leftTable.querySelector('tr');
  firstRow.cells[1].click();
  return firstRow.className;
}"
```

```javascript
// ❌ 錯誤方式：Playwright 點擊 data row
// 數據列 left=-261，element 在視窗外，會被 iframe/dialog 擋住
playwright-cli click e140  // Timeout/Intercept
```

### UI 刪除完整流程（3步驟）

> 所有清單頁的刪除都走同一套流程（企業客戶、聯絡人等）

```javascript
// Step 1: 選中要刪除的行（JS click check cell）
playwright-cli eval "() => {
  const leftTable = document.querySelector('.JuiListLeftTable');
  leftTable.querySelector('tr').cells[1].click();
}"

// Step 2: 點擊刪除按鈕（用 JS 避免被 mask 擋住）
playwright-cli eval "() => {
  const delBtn = document.querySelector('[title=\"刪除當前選中的記錄\"]');
  delBtn.click();
}"

// Step 3: 在 JuiMessageBox 中找 "Determine" 按鈕並點擊
playwright-cli eval "() => {
  const msgBox = document.querySelector('.JuiMessageBox');
  const determineBtn = Array.from(msgBox.querySelectorAll('*'))
    .find(el => el.textContent.trim() === 'Determine');
  determineBtn.click();
}"
```

### 確認對話框：JuiMessageBox 而非 JuiDialog

| 組件 | class | 差異 |
|------|-------|------|
| **刪除確認** | `JuiMessageBox` + `JuiMessageBoxMask` | 樣式不同，按鈕 class 是 `JuiSmallButton` |
| **一般對話框** | `JuiDialog` + `JuiDialogMask` | 舊版表單使用的對話框 |

**確認按鈕文字**：即使 ECP UI 是繁體中文，確認按鈕仍顯示 **"Determine"**（非「確定」）

### 遮罩阻擋問題（Masks Blocking Clicks）

| Mask class | 觸發時機 | 移除方式 |
|------------|---------|---------|
| `JuiMessageBoxMask` | 刪除確認對話框出現時 | `document.querySelectorAll('.JuiMessageBoxMask').forEach(m => m.remove())` |
| `JuiDialogMask` | 浮動對話框出現時 | `document.querySelectorAll('.JuiDialogMask').forEach(m => m.remove())` |
| `JuiMask` | 通用遮罩 | `document.querySelectorAll('.JuiMask').forEach(m => m.remove())` |

**遮罩阻擋症狀**：Playwright 點擊時 Timeout，錯誤訊息 `<div class="JuiMessageBoxMask"> intercepts pointer events`

**解法**：在點擊按鈕前，**先 JS 移除所有 mask**，再用 JS 執行點擊：
```javascript
playwright-cli eval "() => {
  document.querySelectorAll('.JuiMessageBoxMask, .JuiDialogMask, .JuiMask')
    .forEach(m => m.remove());
}"
```

### 一次刪除多筆：必須逐筆執行

- **無批量刪除** — ECP 清單頁沒有 checkbox 全選列，只能單選
- 每刪一筆 → 等待頁面更新 → 選下一筆 → 刪除 → 如此反覆
- 刪除成功後，該列從 DOM 消失，剩下列的 ref 全部重新生成

### 刪除後驗證

```javascript
// 確認還剩 N 筆
playwright-cli eval "() => {
  return document.querySelectorAll('.JuiListDataTable tr').length;
}"

// 確認特定記錄已刪除
playwright-cli eval "() => {
  const text = document.querySelector('.JuiListDataTable').textContent;
  return text.includes('兆豐商業銀行');
}"
```

### 🆕 源碼位置

ECP/JAVA 系統已反編譯的源碼（class 文件）：
- **目錄**: `/home/hch/ecp-source/`
  - `ecp-module-main/` - ECP 業務模組 (2831 class files)
  - `quicksilver-module/` - Quicksilver 核心框架

---

# 🆕 2026-03-31 探索更新：2小時全面頁面發現

> 以下為在**內網測試環境**（`http://10.145.119.234:12821/ecp/`）探索 2 小時的完整發現。
> 所有頁面 URL 均已驗證可用，共 150+ 截圖存於 `/home/hch/ecp-*.png`。

## 雙環境帳號（更新）

| 環境 | URL | 帳號 | 密碼 | 說明 |
|------|-----|------|------|------|
| **外網（正式）** | `https://econtact.ai3.cloud/ecp/` | `James.Huang` | `<ECP_PASSWORD>` | 一般用戶 |
| **內網（測試）** | `http://10.145.119.234:12821/ecp/` | `Administrator` | `<ECP_PASSWORD>` | 系統管理員，可執行 SQL |

---

## ⚠️ 重要發現：側邊欄無法點擊 — 所有頁面 URL 可從 JavaScript 提取

### 問題根因
ECP 側邊選單項目被 CSS `position: absolute; top: -99999px` 藏起來（y 座標在 -20000 以下），Playwright 無法點擊或截取。

### 解決方案：從 `MainFrame._menuMap` 提取所有頁面 URL

```javascript
// 在已登入的 ECP 分頁執行：
browser_run_code(code="async (page) => {
  // 提取所有 238 個頁面 URL
  const menuMap = page.evaluate(() => {
    const results = [];
    const mm = window.Qs?.MainFrame?._menuMap || window.MainFrame?._menuMap;
    if (mm) {
      for (const [k, v] of Object.entries(mm)) {
        if (v?.data?.page) {
          results.push({ menuId: k, page: v.data.page, title: v.data.title });
        }
      }
    }
    return results;
  });
  return menuMap; // 回傳所有 {menuId, page, title}
}")
```

### URL 模式
所有頁面 URL = `http://10.145.119.234:12821/ecp/{page_value}`
其中 `page_value` 來自 `MainFrame._menuMap[menuId].data.page`

### 已驗證的 238 個頁面（按類別）

#### Ecp.* CRM 模組（~100 個）

| 頁面 | URL | 說明 |
|------|-----|------|
| `Ecp.Contact.List.page` | `.../Ecp.Contact.List.page` | 聯絡人列表 |
| `Ecp.Customer.List.page` | `.../Ecp.Customer.List.page` | 企業客戶 |
| `Ecp.Lead.List.page` | `.../Ecp.Lead.List.page` | 線索/商機線索 |
| `Ecp.Opportunity.List.page` | `.../Ecp.Opportunity.List.page` | 商機 |
| `Ecp.Order.List.page` | `.../Ecp.Order.List.page` | 訂單 |
| `Ecp.Contract.List.page` | `.../Ecp.Contract.List.page` | 合約 |
| `Ecp.Receivable.List.page` | `.../Ecp.Receivable.List.page` | 應收帳款 |
| `Ecp.ProceedsCollected.List.page` | `.../Ecp.ProceedsCollected.List.page` | 實際收款 |
| `Ecp.Quote.List.page` | `.../Ecp.Quote.List.page` | 報價 |
| `Ecp.Invoice.List.page` | `.../Ecp.Invoice.List.page` | 發票 |
| `Ecp.ServiceRequest.List.page` | `.../Ecp.ServiceRequest.List.page` | 服務請求 |
| `Ecp.Task.List.page` | `.../Ecp.Task.List.page` | 任務 |
| `Ecp.ProlongTask.List.page` | `.../Ecp.ProlongTask.List.page` | 延時申請-任務 |
| `Ecp.Task.ForeCast.page` | `.../Ecp.Task.ForeCast.page` | 任務指標 |
| `Ecp.Activity.List.page` | `.../Ecp.Activity.List.page` | 活動 |
| `Ecp.Project.List.page` | `.../Ecp.Project.List.page` | 專案 |
| `Ecp.ProjectDelay.List.page` | `.../Ecp.ProjectDelay.List.page` | 延時申請-專案 |
| `Ecp.ProjectMilestoneDelay.List.page` | `.../Ecp.ProjectMilestoneDelay.List.page` | 里程碑延時 |
| `Ecp.ServiceRequestDelay.List.page` | `.../Ecp.ServiceRequestDelay.List.page` | 服務請求延時 |
| `Ecp.ServiceRequest.ForeCast.page` | `.../Ecp.ServiceRequest.ForeCast.page` | 服務請求指標 |
| `Ecp.Product.List.page` | `.../Ecp.Product.List.page` | 產品 |
| `Ecp.ProductCatalog.List.page` | `.../Ecp.ProductCatalog.List.page` | 產品目錄 |
| `Ecp.PriceList.List.page` | `.../Ecp.PriceList.List.page` | 價格表 |
| `Ecp.Unit.List.page` | `.../Ecp.Unit.List.page` | 單位 |
| `Ecp.Expense.List.page` | `.../Ecp.Expense.List.page` | 費用 |
| `Ecp.HelpDesk.List.page` | `.../Ecp.HelpDesk.List.page` | 服務台 |
| `Ecp.WorkSheet.List.page` | `.../Ecp.WorkSheet.List.page` | 工作檢查表 |
| `Ecp.Defect.List.page` | `.../Ecp.Defect.List.page` | 產品缺陷 |
| `Ecp.Repairment.List.page` | `.../Ecp.Repairment.List.page` | 維修 |
| `Ecp.Vendor.List.page` | `.../Ecp.Vendor.List.page` | 供應商 |
| `Ecp.Competitor.List.page` | `.../Ecp.Competitor.List.page` | 競爭對手 |
| `Ecp.Phone.List.page` | `.../Ecp.Phone.List.page` | 電話 |
| `Ecp.CallLog.List.page` | `.../Ecp.CallLog.List.page` | 通話記錄 |
| `Ecp.CallLog.CtiInbound.page` | `.../Ecp.CallLog.CtiInbound.page` | 類比來話 |
| `Ecp.CallLogAudioKey.List.page` | `.../Ecp.CallLogAudioKey.List.page` | 通話錄音密鑰 |
| `Ecp.PhonePlan.List.page` | `.../Ecp.PhonePlan.List.page` | 電話計劃 |
| `Ecp.SupportMail.List.page` | `.../Ecp.SupportMail.List.page` | 待處理郵件 |
| `Ecp.SupportMailError.List.page` | `.../Ecp.SupportMailError.List.page` | 客訴郵件 |
| `Ecp.Email.List.page` | `.../Ecp.Email.List.page` | 郵件 |
| `Ecp.Email.Form.page` | `.../Ecp.Email.Form.page` | 撰寫郵件 |
| `Ecp.EmailTemplate.List.page` | `.../Ecp.EmailTemplate.List.page` | 郵件範本 |
| `Ecp.Survey.List.page` | `.../Ecp.Survey.List.page` | 問卷 |
| `Ecp.MarketPlan.List.page` | `.../Ecp.MarketPlan.List.page` | 行銷計劃 |
| `Ecp.RefusedMarketing.List.page` | `.../Ecp.RefusedMarketing.List.page` | 行銷黑名單 |
| `Ecp.TargetCustomer.List.page` | `.../Ecp.TargetCustomer.List.page` | 目標客戶 |
| `Ecp.TargetCustomerContactMapping.List.page` | `.../Ecp.TargetCustomerContactMapping.List.page` | 目標客戶欄位對應 |
| `Ecp.TargetCustomerContactStatus.List.page` | `.../Ecp.TargetCustomerContactStatus.List.page` | 目標客戶接觸狀態 |
| `Ecp.TargetCustomerActivityCatalog.List.page` | `.../Ecp.TargetCustomerActivityCatalog.List.page` | 目標客戶互動類型目錄 |
| `Ecp.TargetCustomerActivityCode.List.page` | `.../Ecp.TargetCustomerActivityCode.List.page` | 目標客戶互動項目 |
| `Ecp.Document.List.page` | `.../Ecp.Document.List.page` | 文件 |
| `Ecp.Knowledge.List.page` | `.../Ecp.Knowledge.List.page` | 知識庫 |
| `Ecp.ProcessKnowledge.List.page` | `.../Ecp.ProcessKnowledge.List.page` | 流程知識 |
| `Ecp.KnowledgeCatalog.List.page` | `.../Ecp.KnowledgeCatalog.List.page` | 知識目錄 |
| `Ecp.DocumentCatalog.List.page` | `.../Ecp.DocumentCatalog.List.page` | 文件目錄 |
| `Ecp.SynonymCatalog.List.page` | `.../Ecp.SynonymCatalog.List.page` | 同義詞目錄 |
| `Ecp.MeetingMinutes.List.page` | `.../Ecp.MeetingMinutes.List.page` | 會議記錄 |
| `Ecp.TaskMethod.List.page` | `.../Ecp.TaskMethod.List.page` | 任務方法 |
| `Ecp.TRStandardGist.List.page` | `.../Ecp.TRStandardGist.List.page` | 工時日誌標準摘要 |
| `Ecp.CalendarSetup.List.page` | `.../Ecp.CalendarSetup.List.page` | 日曆設定 |
| `Ecp.Schedule.List.page` | `.../Ecp.Schedule.List.page` | 日程 |
| `Ecp.Holiday.List.page` | `.../Ecp.Holiday.List.page` | 假期 |
| `Ecp.TimeReport.List.page` | `.../Ecp.TimeReport.List.page` | 工時日誌 |
| `Ecp.Discuss.List.page` | `.../Ecp.Discuss.List.page` | 討論區 |
| `Ecp.Currency.List.page` | `.../Ecp.Currency.List.page` | 幣別 |
| `Ecp.ImageText.List.page` | `.../Ecp.ImageText.List.page` | 圖文訊息 |
| `Ecp.PushLog.List.page` | `.../Ecp.PushLog.List.page` | 推播紀錄 |
| `Ecp.MarketContent.List.page` | `.../Ecp.MarketContent.List.page` | 行銷文案 |
| `Ecp.SaleWay.List.page` | `.../Ecp.SaleWay.List.page` | 銷售方式 |
| `Ecp.OpportunitySetup.List.page` | `.../Ecp.OpportunitySetup.List.page` | 商機配置 |
| `Ecp.ServiceRequestMethod.List.page` | `.../Ecp.ServiceRequestMethod.List.page` | 服務請求方法 |
| `Ecp.ActivityCatalog.List.page` | `.../Ecp.ActivityCatalog.List.page` | 活動目錄 |
| `Ecp.ActivityCode.List.page` | `.../Ecp.ActivityCode.List.page` | 活動代碼 |
| `Ecp.ProjectMethod.List.page` | `.../Ecp.ProjectMethod.List.page` | 專案方法 |
| `Ecp.SkillSetup.List.page` | `.../Ecp.SkillSetup.List.page` | 技能設定 |
| `Ecp.WgConfig.List.page` | `.../Ecp.WgConfig.List.page` | 群組設定 |
| `Ecp.DidConfig.List.page` | `.../Ecp.DidConfig.List.page` | DID 設定 |
| `Ecp.PromptConfig.List.page` | `.../Ecp.PromptConfig.List.page` | Prompt 設定 |
| `Ecp.PromptField.prepareConfigList.page` | `.../Ecp.PromptField.prepareConfigList.page` | Prompt 欄位設定 |
| `Ecp.MoField.prepareConfigList.page` | `.../Ecp.MoField.prepareConfigList.page` | MO 欄位設定 |
| `Ecp.SupportMailGroup.List.page` | `.../Ecp.SupportMailGroup.List.page` | 支援郵件群組 |
| `Ecp.ContactCollectionSetup.List.page` | `.../Ecp.ContactCollectionSetup.List.page` | 聯絡人摘要設定 |
| `Ecp.NormalizePhone.List.page` | `.../Ecp.NormalizePhone.List.page` | 電話號碼正規化 |
| `Ecp.Contact.OutlookMappingConfig.page` | `.../Ecp.Contact.OutlookMappingConfig.page` | 聯絡人匯出設定 |
| `Ecp.ContactLabel.List.page` | `.../Ecp.ContactLabel.List.page` | 標籤 |
| `Ecp.ContactNickname.List.page` | `.../Ecp.ContactNickname.List.page` | 暱稱 |
| `Ecp.ContactInteraction.List.page` | `.../Ecp.ContactInteraction.List.page` | 互動記錄 |
| `Ecp.ContactCollection.List.page` | `.../Ecp.ContactCollection.List.page` | 聯絡人摘要 |
| `Ecp.ContactGroup.List.page` | `.../Ecp.ContactGroup.List.page` | 聯絡人群組 |
| `Ecp.EmailAddress.List.page` | `.../Ecp.EmailAddress.List.page` | Email 地址 |
| `Ecp.Address.List.page` | `.../Ecp.Address.List.page` | 地址 |
| `Ecp.ObjectContact.List.page` | `.../Ecp.ObjectContact.List.page` | 物件聯絡人 |
| `Ecp.ContactChatPushLog.List.page` | `.../Ecp.ContactChatPushLog.List.page` | 聊天推送紀錄 |
| `Ecp.ApPage.List.page` | `.../Ecp.ApPage.List.page` | AP 分頁 |
| `Ecp.EcpPicture.List.page` | `.../Ecp.EcpPicture.List.page` | 圖片管理 |
| `Ecp.UnitDiscussConfig.List.page` | `.../Ecp.UnitDiscussConfig.List.page` | 單位討論設定 |
| `Ecp.ChannelNotificationType.List.page` | `.../Ecp.ChannelNotificationType.List.page` | 多管道通知類型 |
| `Ecp.ChannelNotificationStatus.List.page` | `.../Ecp.ChannelNotificationStatus.List.page` | 管道通知狀態 |
| `Ecp.TenantChannelProvider.List.page` | `.../Ecp.TenantChannelProvider.List.page` | 租戶管道供應商 |
| `Ecp.TenantChannel.List.page` | `.../Ecp.TenantChannel.List.page` | 租戶管道 |

#### Ecp.* 聊天/智能服務模組

| 頁面 | URL | 說明 |
|------|-----|------|
| `Ecp.ChatWorkGroup.List.page` | `.../Ecp.ChatWorkGroup.List.page` | 文字客服群組 |
| `Ecp.ChatBoxMessageCatalog.List.page` | `.../Ecp.ChatBoxMessageCatalog.List.page` | 罐頭訊息目錄 |
| `Ecp.ChatBoxMessage.List.page` | `.../Ecp.ChatBoxMessage.List.page` | 罐頭訊息 |
| `Ecp.ChatAdvertisingInformation.List.page` | `.../Ecp.ChatAdvertisingInformation.List.page` | 廣告訊息 |
| `Ecp.Chat.ChatSensitiveWord.List.page` | `.../Ecp.Chat.ChatSensitiveWord.List.page` | 敏感詞設定 |
| `Ecp.Chat.ChatSensitiveMonitor.List.page` | `.../Ecp.Chat.ChatSensitiveMonitor.List.page` | 敏感詞監控 |
| `Ecp.Chat.ChatSensitiveKeyword.List.page` | `.../Ecp.Chat.ChatSensitiveKeyword.List.page` | 敏感詞關鍵字 |
| `Ecp.RoomStatusLog.List.page` | `.../Ecp.RoomStatusLog.List.page` | 聊天室狀態紀錄 |
| `Ecp.ChatLog.List.page` | `.../Ecp.ChatLog.List.page` | 聊天紀錄 |
| `Ecp.ChatRoomListSetting.List.page` | `.../Ecp.ChatRoomListSetting.List.page` | 聊天室列表設定 |
| `Ecp.ChatKeep.List.page` | `.../Ecp.ChatKeep.List.page` | 服務對象綁定 |
| `Ecp.ExpressChatMaster.List.page` | `.../Ecp.ExpressChatMaster.List.page` | 服務總機管理 |
| `Ecp.ExpressChatParameter.List.page` | `.../Ecp.ExpressChatParameter.List.page` | 參數文案管理 |
| `Ecp.ChatRequeueLog.List.page` | `.../Ecp.ChatRequeueLog.List.page` | 聊天重新排程紀錄 |
| `Ecp.MonitorAgentChatRoom.List.page` | `.../Ecp.MonitorAgentChatRoom.List.page` | 客服監控 |
| `Ecp.OfflineRoom.List.page` | `.../Ecp.OfflineRoom.List.page` | 訊息聊天室 |
| `Ecp.FbFans.List.page` | `.../Ecp.FbFans.List.page` | FB 粉絲 |
| `Ecp.FbPost.List.page` | `.../Ecp.FbPost.List.page` | FB 貼文 |
| `Ecp.FbBanUser.List.page` | `.../Ecp.FbBanUser.List.page` | FB 黑名單 |
| `Ecp.FbFanPageManage.List.page` | `.../Ecp.FbFanPageManage.List.page` | FB 粉絲頁管理 |
| `Ecp.CrawlerSchedule.List.page` | `.../Ecp.CrawlerSchedule.List.page` | 社群排程 |
| `Ecp.AskGptTask.List.page` | `.../Ecp.AskGptTask.List.page` | AI 問答任務 |
| `Ecp.AskGptResult.List.page` | `.../Ecp.AskGptResult.List.page` | AI 問答結果 |
| `Ecp.KnowledgeTrainTask.List.page` | `.../Ecp.KnowledgeTrainTask.List.page` | Copilot 訓練進度 |
| `Ecp.ImportKmFileProgress.List.page` | `.../Ecp.ImportKmFileProgress.List.page` | 知識匯入進度 |
| `Ecp.CopilotGreeting.List.page` | `.../Ecp.CopilotGreeting.List.page` | Copilot 招呼語 |
| `Ecp.AIRoom.List.page` | `.../Ecp.AIRoom.List.page` | AI 對話服務 |
| `Ecp.AIMessage.List.page` | `.../Ecp.AIMessage.List.page` | AI 對話紀錄 |
| `Ecp.CopilotMessage.List.page` | `.../Ecp.CopilotMessage.List.page` | Copilot 對話紀錄 |
| `Ecp.CopilotBasicQuestion.List.page` | `.../Ecp.CopilotBasicQuestion.List.page` | 驗證問題 |
| `Ecp.CopilotQuestionList.List.page` | `.../Ecp.CopilotQuestionList.List.page` | 多輪對話 |
| `Ecp.CopilotVerifyList.List.page` | `.../Ecp.CopilotVerifyList.List.page` | 驗證任務 |
| `Ecp.KnowledgeQa.List.page` | `.../Ecp.KnowledgeQa.List.page` | 知識試題產生 |
| `Ecp.Prompt.List.page` | `.../Ecp.Prompt.List.page` | AI 提詞器 |
| `Ecp.SubPrompt.List.page` | `.../Ecp.SubPrompt.List.page` | AI 子提詞 |
| `Ecp.PromptArgs.List.page` | `.../Ecp.PromptArgs.List.page` | AI 提詞參數 |
| `Ecp.PromptInputTemplate.List.page` | `.../Ecp.PromptInputTemplate.List.page` | AI 資料來源範本 |
| `Ecp.CopilotBotFeature.List.page` | `.../Ecp.CopilotBotFeature.List.page` | Copilot 機器人特徵 |
| `Ecp.AiAgentEntity.List.page` | `.../Ecp.AiAgentEntity.List.page` | AI Agent 實體 |
| `Ecp.AiAgentFunction.List.page` | `.../Ecp.AiAgentFunction.List.page` | AI Agent 任務 |
| `Ecp.LLMApiControl.List.page` | `.../Ecp.LLMApiControl.List.page` | GPT 相關設定 |
| `Ecp.LLMParameterTemplate.List.page` | `.../Ecp.LLMParameterTemplate.List.page` | GPT 參數模板 |
| `Ecp.ExploreMap.List.page` | `.../Ecp.ExploreMap.List.page` | 探索地圖 |
| `Ecp.LLMWebSearch.List.page` | `.../Ecp.LLMWebSearch.List.page` | Web Search 管理 |
| `Ecp.IAAnswerOptimization.List.page` | `.../Ecp.IAAnswerOptimization.List.page` | IA 答案優化 |
| `Ecp.RelatedKnowledge.List.page` | `.../Ecp.RelatedKnowledge.List.page` | 相關知識 |
| `Ecp.SttTask.List.page` | `.../Ecp.SttTask.List.page` | STT 轉換任務 |
| `Ecp.STTTaskRecord.List.page` | `.../Ecp.STTTaskRecord.List.page` | 離線語音轉文字任務 |
| `Ecp.VideoSTTTaskRecord.List.page` | `.../Ecp.VideoSTTTaskRecord.List.page` | 視訊客服轉文字任務 |
| `Ecp.VideoCapture.List.page` | `.../Ecp.VideoCapture.List.page` | 視訊影像擷取 |

#### Ecp.* 質檢/考試模組

| 頁面 | URL | 說明 |
|------|-----|------|
| `Ecp.QMItem.List.page` | `.../Ecp.QMItem.List.page` | 質檢資料 |
| `Ecp.QMTaskDetail.List.page` | `.../Ecp.QMTaskDetail.List.page` | 質檢結果 |
| `Ecp.QMTaskDetail.CheckList.page` | `.../Ecp.QMTaskDetail.CheckList.page` | 覆核項目 |
| `Ecp.QMPlan.List.page` | `.../Ecp.QMPlan.List.page` | 質檢計劃 |
| `Ecp.QMTask.List.page` | `.../Ecp.QMTask.List.page` | 質檢任務 |
| `Ecp.QMTask.QMTaskExecuteList.page` | `.../Ecp.QMTask.QMTaskExecuteList.page` | 質檢執行明細 |
| `Ecp.QMScoreSheet.List.page` | `.../Ecp.QMScoreSheet.List.page` | 質檢規則表 |
| `Ecp.QMDetailDictionary.List.page` | `.../Ecp.QMDetailDictionary.List.page` | 質檢項目對應文字 |
| `Ecp.QMTaskExecuteControl.List.page` | `.../Ecp.QMTaskExecuteControl.List.page` | 質檢任務執行管理 |
| `Ecp.Question.List.page` | `.../Ecp.Question.List.page` | 試題 |
| `Ecp.ExamPaper.List.page` | `.../Ecp.ExamPaper.List.page` | 試卷 |
| `Ecp.Exam.List.page` | `.../Ecp.Exam.List.page` | 考試 |
| `Ecp.Exam.StudentList.page` | `.../Ecp.Exam.StudentList.page` | 考試学员 |
| `Ecp.AnswerSheet.List.page` | `.../Ecp.AnswerSheet.List.page` | 答案卷 |
| `Ecp.QuestionTag.List.page` | `.../Ecp.QuestionTag.List.page` | 試題標籤 |
| `Ecp.MessageTemplate.List.page` | `.../Ecp.MessageTemplate.List.page` | 簡訊範本 |

#### Qs.* 系統管理模組（~60 個）

| 頁面 | URL | 說明 |
|------|-----|------|
| `Qs.User.List.page` | `.../Qs.User.List.page` | 員工 |
| `Qs.Department.List.page` | `.../Qs.Department.List.page` | 部門 |
| `Qs.Role.List.page` | `.../Qs.Role.List.page` | 角色 |
| `Qs.Account.List.page` | `.../Qs.Account.List.page` | 帳號 |
| `Qs.OnlineUser.List.page` | `.../Qs.OnlineUser.List.page` | 在線用戶 |
| `Qs.LoginLog.MyList.page` | `.../Qs.LoginLog.MyList.page` | 我的登入日誌 |
| `Qs.LoginLog.List.page` | `.../Qs.LoginLog.List.page` | 登入日誌 |
| `Qs.BusinessLog.List.page` | `.../Qs.BusinessLog.List.page` | 業務日誌 |
| `Qs.Monitor.Performance.page` | `.../Qs.Monitor.Performance.page` | 效能監控 |
| `Qs.Monitor.Cache.page` | `.../Qs.Monitor.Cache.page` | 快取監控 |
| `Qs.DiskFile.ConverterMonitor.page` | `.../Qs.DiskFile.ConverterMonitor.page` | 附件轉換監控 |
| `Qs.SqlUpgradeLog.List.page` | `.../Qs.SqlUpgradeLog.List.page` | SQL 更新日誌 |
| `Qs.SqlExecuteLog.List.page` | `.../Qs.SqlExecuteLog.List.page` | SQL 執行日誌 |
| `Qs.EventLog.List.page` | `.../Qs.EventLog.List.page` | 事件日誌 |
| `Qs.Parameter.System.page` | `.../Qs.Parameter.System.page` | 系統參數 |
| `Qs.Parameter.Company.page` | `.../Qs.Parameter.Company.page` | 公司參數（可能Error） |
| `Qs.ParameterDefinition.List.page` | `.../Qs.ParameterDefinition.List.page` | 參數定義 |
| `Qs.Menu.List.page` | `.../Qs.Menu.List.page` | 選單 |
| `Qs.IconMenu.List.page` | `.../Qs.IconMenu.List.page` | 圖示選單 |
| `Qs.Privilege.List.page` | `.../Qs.Privilege.List.page` | 權限 |
| `Qs.RelationPrivilege.List.page` | `.../Qs.RelationPrivilege.List.page` | 關聯權限 |
| `Qs.SlavePagePrivilege.List.page` | `.../Qs.SlavePagePrivilege.List.page` | 子頁面權限 |
| `Qs.TextResource.List.page` | `.../Qs.TextResource.List.page` | 文字資源 |
| `Qs.Unit.List.page` | `.../Qs.Unit.List.page` | 單元 |
| `Qs.SerialNumber.List.page` | `.../Qs.SerialNumber.List.page` | 流水號 |
| `Qs.UnitConvert.List.page` | `.../Qs.UnitConvert.List.page` | 單位轉換 |
| `Qs.Page.List.page` | `.../Qs.Page.List.page` | 頁面 |
| `Qs.SpecialPath.Config.page` | `.../Qs.SpecialPath.Config.page` | 特殊路徑設定 |
| `Qs.TokenConfig.List.page` | `.../Qs.TokenConfig.List.page` | Token 設定 |
| `Qs.RemoteApiGroup.List.page` | `.../Qs.RemoteApiGroup.List.page` | 遠端 API 群組 |
| `Qs.RemoteApi.List.page` | `.../Qs.RemoteApi.List.page` | 遠端 API |
| `Qs.LocalApiGroup.List.page` | `.../Qs.LocalApiGroup.List.page` | 本地 API 群組 |
| `Qs.LocalApi.List.page` | `.../Qs.LocalApi.List.page` | 本地 API |
| `Qs.WebServiceProvider.List.page` | `.../Qs.WebServiceProvider.List.page` | WebService 提供者 |
| `Qs.Language.List.page` | `.../Qs.Language.List.page` | 語言 |
| `Qs.Dictionary.List.page` | `.../Qs.Dictionary.List.page` | 字典 |
| `Qs.Picture.List.page` | `.../Qs.Picture.List.page` | 圖片 |
| `Qs.DataLink.List.page` | `.../Qs.DataLink.List.page` | 資料連結 |
| `Qs.SystemEmail.List.page` | `.../Qs.SystemEmail.List.page` | 系統郵件 |
| `Qs.ShortMessage.List.page` | `.../Qs.ShortMessage.List.page` | 簡訊 |
| `Qs.Notice.List.page` | `.../Qs.Notice.List.page` | 通知 |
| `Qs.Timer.List.page` | `.../Qs.Timer.List.page` | 計時器 |
| `Qs.TimerLog.List.page` | `.../Qs.TimerLog.List.page` | 計時器日誌 |
| `Qs.Deputy.List.page` | `.../Qs.Deputy.List.page` | 代理人 |
| `Qs.Chart.Center.page` | `.../Qs.Chart.Center.page` | 圖表中心 |
| `Qs.Chart.List.page` | `.../Qs.Chart.List.page` | 圖表設定 |
| `Qs.Report.Center.page` | `.../Qs.Report.Center.page` | 報表中心 |
| `Qs.Report.List.page` | `.../Qs.Report.List.page` | 報表設定 |
| `Qs.Bill.List.page` | `.../Qs.Bill.List.page` | 單據 |
| `Qs.Script.CommonList.page` | `.../Qs.Script.CommonList.page` | 通用腳本 |
| `Qs.Misc.ApplicationManagement.page` | `.../Qs.Misc.ApplicationManagement.page` | 應用管理 |
| `Qs.ClusterNode.List.page` | `.../Qs.ClusterNode.List.page` | 叢集節點 |
| `Qs.Misc.DatabaseIndexManage.page` | `.../Qs.Misc.DatabaseIndexManage.page` | 索引管理 |
| `Qs.DatabaseMark.List.page` | `.../Qs.DatabaseMark.List.page` | 資料庫標記 |
| `Qs.Export.SqlExport.page` | `.../Qs.Export.SqlExport.page` | SQL 匯出（可能Error） |
| `Qs.SystemTool.SqlExecute.page` | `.../Qs.SystemTool.SqlExecute.page` | SQL 執行（需Administrator） |
| `Qs.Misc.CorsConfig.page` | `.../Qs.Misc.CorsConfig.page` | CORS 跨域設定 |
| `Qs.DirectApiGroup.List.page` | `.../Qs.DirectApiGroup.List.page` | Direct API 群組 |
| `Qs.OuterWebSocketRegion.List.page` | `.../Qs.OuterWebSocketRegion.List.page` | WebSocket 外部設定 |
| `Qs.MobileMenu.List.page` | `.../Qs.MobileMenu.List.page` | 手機選單 |
| `Qs.HomepageItem.UserHomepageConfig.page` | `.../Qs.HomepageItem.UserHomepageConfig.page` | 使用者首頁設定 |
| `Qs.HomepageItem.SystemHomepageConfig.page` | `.../Qs.HomepageItem.SystemHomepageConfig.page` | 系統首頁設定 |
| `Qs.GraphHomepage.Config.page` | `.../Qs.GraphHomepage.Config.page` | 圖形首頁設定 |
| `Qs.User.Workbench.page` | `.../Qs.User.Workbench.page` | 工作檯設定 |
| `Qs.Picture.MyAvatar.page` | `.../Qs.Picture.MyAvatar.page` | 我的頭像 |

#### Wf.* 工作流模組

| 頁面 | URL | 說明 |
|------|-----|------|
| `Wf.WorkItem.List.page` | `.../Wf.WorkItem.List.page` | 工作項 |
| `Wf.Process.List.page` | `.../Wf.Process.List.page` | 流程實例 |
| `Wf.Workflow.List.page` | `.../Wf.Workflow.List.page` | 流程定義 |

#### Ecp.* 銷售預測

| 頁面 | URL | 說明 |
|------|-----|------|
| `Ecp.SaleForecast.Forecast.page` | `.../Ecp.SaleForecast.Forecast.page` | 銷售預測 |
| `Ecp.SaleForecast.Company.page` | `.../Ecp.SaleForecast.Company.page` | 公司銷售計劃 |
| `Ecp.SaleForecast.Department.page` | `.../Ecp.SaleForecast.Department.page` | 部門銷售計劃分解 |

#### KMCopilot 知識管理

| 頁面 | URL | 說明 |
|------|-----|------|
| `KMCopilot/main/main.html` | `.../KMCopilot/main/main.html` | Copilot 主頁 |
| `KMCopilot/webView/webView.html` | `.../KMCopilot/webView/webView.html` | Copilot WebView |
| `KMCopilot/chat/chat.html` | `.../KMCopilot/chat/chat.html` | Copilot Chat |

---

### 無法直接訪問的頁面（已確認）

這些頁面存在於選單但無法直接 URL 訪問（需要特殊上下文）：
- `Ecp.ProductIssue.List.page` — API 存在但無頁面
- `Ecp.Robot.List.page` — "no unit encoding" Error
- `Ecp.QAIssue.List.page` — "no unit encoding" Error
- `Ecp.VerificationQuestion.List.page` — "no unit encoding" Error
- `Ecp.DailyTask.List.page` — "no unit encoding" Error
- `Ecp.WorkOrder.List.page` — "no unit encoding" Error
- `Ecp.ChartCenter.List.page` — "no unit encoding" Error
- `Ecp.Staff.List.page` / `Ecp.Employee.List.page` — 需從功能表進入
- `Qs.Dashboard.page` / `Qs.Diary.page` — 需特殊上下文
- `Qs.SqlLog.page` / `Qs.ProcessDef.List.page` — 需特殊上下文
- `Ecp.CustomerSelfService.portal` / `Ecp.AgentSelfService.portal` — 404

### Chat Supervisor 特殊頁面（需 WebSocket 上下文）

| 頁面 | URL | 說明 |
|------|-----|------|
| `Ecp.ChatSupervisor.page` | `.../Ecp.ChatSupervisor.page` | 文字客服監控 |
| `Ecp.ChatRoom.Chat.page` | `.../Ecp.ChatRoom.Chat.page` | 文字客服聊天室 |
| `Ecp.ChatSupervisor.getEcpChatSupervisorParameters.data` | API | 監控參數 |
| `Ecp.ChatSupervisor.getUnreadyStatus.data` | API | 未就緒狀態 |

---

## 快速導航：直接 URL 訪問（無需側邊選單點擊）

所有頁面均可直接用 `playwright-cli goto <url>` 訪問，格式：
```text
playwright-cli goto http://10.145.119.234:12821/ecp/{PageValue}
```

例如：
```text
playwright-cli goto http://10.145.119.234:12821/ecp/Ecp.Contact.List.page
playwright-cli goto http://10.145.119.234:12821/ecp/Wf.WorkItem.List.page
playwright-cli goto http://10.145.119.234:12821/ecp/Qs.User.List.page
```

---



## 概述

自動登入 ECP 並執行任何操作：查詢、新增員工、設定帳號密碼、出勤、工時等。

## 工具需求

**必須使用 `playwright-cli`**（非 Playwright MCP）：

| 動作 | playwright-cli 命令 |
|------|-------------------|
| 開啟瀏覽器並導航 | `playwright-cli open <url>` 或 `playwright-cli goto <url>` |
| 頁面快照（取 ref） | `playwright-cli snapshot` |
| 截圖確認 | `playwright-cli screenshot` |
| 點擊元素 | `playwright-cli click <ref>` |
| 填寫輸入框 | `playwright-cli fill <ref> <text>` |
| 執行 JS | `playwright-cli run-code <code>` |
| 處理對話框 | `playwright-cli dialog-accept` |
| 新分頁 | `playwright-cli tab-new [url]` |
| 關閉分頁 | `playwright-cli tab-close <index>` |

---

## ⚠️ 重要注意事項

### 離開頁面確認對話框（⚠️ 最常見問題）

ECP 離開任何有表單的頁面都會彈出瀏覽器原生 `beforeunload` 確認視窗（「要離開網站嗎？」）。

**✅ 根本解法：見 Step 1d — 開新分頁並用 `addInitScript` 攔截 `EventTarget.prototype.addEventListener`，永久阻止 beforeunload 被設定。**

#### ❌ 廢棄做法（不穩定，不要用）
```text
browser_handle_dialog(accept=true)          // 時機不對常常失敗
page.once('dialog', async d => d.accept())  // 只能處理單次，仍會被卡住
```

### 已重複登入確認
若系統提示「您已經登入本系統，是否確認繼續登入並斷線其它登入？」，點擊確定繼續。

### 選單 ref 每次會話會變動
Step 3 的 ref 編號（如 `e67`, `e481`）僅供參考，實際 ref 需要 `browser_snapshot()` 後再確認。**建議使用 `browser_run_code` 的文字比對方式導航**（見下方）。

---

## Step 1：登入 ECP

> ⚠️ **必須按此順序**：先開新分頁裝 patch，再登入。順序錯誤會導致 `beforeunload` 對話框無法攔截。

### 完整登入流程（一次完成）

```bash
# Step 1: 開新分頁（patch 只對新分頁有效）
playwright-cli tab-new

# Step 2: 使用 initScript 攔截 beforeunload（透過設定檔或環境变量）
# 方式 A：使用 initScript 設定檔攔截 beforeunload
# 創建 init-script.js:
#   const _orig = EventTarget.prototype.addEventListener;
#   EventTarget.prototype.addEventListener = function(type, listener, options) {
#     if (type === 'beforeunload') return;
#     return _orig.call(this, type, listener, options);
#   };
#   Object.defineProperty(window, 'onbeforeunload', {
#     set: () => {}, get: () => null, configurable: true
#   });

# Step 3: 導航到登入頁
playwright-cli goto https://econtact.ai3.cloud/ecp/Qs.OnlineUser.Login.page

# Step 4: 填寫登入資訊（使用 fill 命令，playwright-cli 會自動找到元素）
playwright-cli fill input[type="text"] James.Huang
playwright-cli fill input[type="password"] <ECP_PASSWORD>
playwright-cli click "登 入"

# Step 5: 處理「您已經登入本系統」多重登入對話框
# 如果看到重複登入提示，使用 snapshot 並點擊確定
playwright-cli snapshot
playwright-cli click "確定"
```

> ✅ 執行後即可自由導航、關閉頁簽、切換頁面，不再有任何 beforeunload 對話框。
> ❌ 舊分頁（已開著的）無法補打 patch，必須重開新分頁。

### 帳號速查

| 帳號 | 密碼 | 說明 |
|------|------|------|
| `James.Huang` | `<ECP_PASSWORD>` | 一般用戶（外網，正式） |
| `Administrator` | `<ECP_PASSWORD>` | 系統管理員（內網，測試） |

---

## Step 2：導航到目標頁面

### 方法 A：用文字比對點擊（最可靠）

```bash
# 導航到「系統管理 → 組織架構 → 員工」
# 使用 click 搭配文字定位
playwright-cli click "系統管理"
playwright-cli click "組織架構"
playwright-cli click "員工"
```

### 方法 B：直接用 ref 點擊（需先 snapshot 確認 ref）
```bash
playwright-cli snapshot
# 找到選單 ref 後點擊
playwright-cli click eXXX
```

### 方法 C：直接導航（部分頁面在系統管理外無法直接訪問）
```bash
playwright-cli goto https://econtact.ai3.cloud/ecp/Qs.Employee.List.page
# 注意：此頁面需在主框架內開啟，直接導航可能無法正常顯示
```

---

## 📋 完整選單 → Page URL 對應表

### 辦公自動化

| 選單名稱 | Page URL（相對路徑） | 說明 |
|----------|---------------------|------|
| 首頁 | `Qs.Homepage.page` | ECP 主首頁 |
| 同事 | `Ecp.Colleague.List.page` | 同事通訊錄 |
| 聯絡人 | `Ecp.Contact.List.page?args=%7B%22schemaId%22%3A%22d3e8a3df-6be1-4b59-b78f-46fb8363e26b%22%7D` | 外部聯絡人 |
| 申請單 | `CUS.ApplicationForm.List.page` | 各類申請單 |
| 文檔 | `Ecp.Document.List.page` | 文件管理 |
| 知識 | `Ecp.Knowledge.List.page?args=%7B%22hasQuerySchemaBox%22%3Afalse%7D` | 知識庫 |
| 公告 | `Ecp.Proclamation.List.page?args=%7B%22hasQuerySchemaBox%22%3Afalse%7D` | 系統公告 |
| 郵件 | `Ecp.Email.List.page` | 收件匣 |
| 草稿 | `Ecp.Email.List.page?args=%7B%22hasQuerySchemaBox%22%3Afalse%2C%22schemaId%22%3A%22571f8562-b978-43fe-b8ce-52fd20153aee%22%7D` | 郵件草稿 |
| 工作項 | `Wf.WorkItem.List.page?args=%7B%22hasQuerySchemaBox%22%3Afalse%2C%22schemaId%22%3A%22244ec586-31d8-4236-9a92-cd54d73ba449%22%7D` | 工作項清單 |
| 流程 | `Wf.Process.List.page?args=%7B%22hasQuerySchemaBox%22%3Afalse%7D` | 工作流程 |
| 任務 | `Ecp.Task.List.page?args=%7B%22hideLeftZone%22%3Atrue%7D` | 任務清單 |
| 費用 | `Ecp.Expense.List.page` | 費用申請 |
| 費用明細 | `Ecp.ExpenseDetail.List.page` | 費用明細 |
| 工時日誌日曆 | `Ecp.Calendar.page?args=%7B%22hideLeftZone%22%3Atrue%2C%22type%22%3A%22timeReportDetail%22%2C%22value%22%3A%22Ecp.TimeReportDetail%22%7D` | 工時日誌月曆 |
| 我未結案的服務台 | `Ecp.HelpDesk.List.page` | 未結案服務台 |
| 出勤打卡 | `Ecp.CheckIn.List.page?args=%7B%22hideLeftZone%22%3Atrue%7D` | 出勤打卡記錄 |
| 請假單 | `Ecp.LeavePermit.List.page` | 請假申請 |
| 日曆 | `Ecp.Calendar.page` | 個人日曆 |

### 客戶關係

| 選單名稱 | Page URL（相對路徑） | 說明 |
|----------|---------------------|------|
| 企業 | `Ecp.Customer.List.page` | 企業客戶清單 |
| 問卷 | `Ecp.Survey.List.page` | 問卷管理 |
| 服務請求 | `Ecp.ServiceRequest.List.page?args=%7B%22schemaId%22%3A%224eb82794-b63c-4d1a-ab66-0a68cf817167%22%7D` | 服務請求主清單 |
| 服務請求-執行 | `Ecp.ServiceRequest.List.page?args=%7B%22schemaId%22%3A%22e156203c-1566-48e9-8ccb-e15835007890%22%7D` | 執行中 |
| 服務請求-草稿 | `Ecp.ServiceRequest.List.page?args=%7B%22schemaId%22%3A%225518c5da-cf45-49dc-b03d-1ca9694e8806%22%7D` | 草稿 |
| 服務請求-審核 | `Ecp.ServiceRequest.List.page?args=%7B%22schemaId%22%3A%2216076cfb-6960-0596-6540-c65444558360%22%7D` | 待審核 |
| 延時申請-服務請求 | `Ecp.ServiceRequestDelay.List.page` | 服務請求延時申請 |
| 專案 | `Ecp.Project.List.page` | 專案清單 |
| 專案里程碑 | `Ecp.ProjectMilestoned.List.page` | 里程碑清單 |
| 延時申請-專案 | `Ecp.ProjectDelay.List.page` | 專案延時申請 |
| 延時申請-專案里程碑 | `Ecp.ProjectMilestoneDelay.List.page` | 里程碑延時申請 |
| 專案審核逾期 | `Ecp.ProjectAgreeAlert.List.page` | 審核逾期警示 |
| 專案里程碑逾期 | `Ecp.ProjectMileStoneAlert.List.page` | 里程碑逾期警示 |
| 工作項目審核逾期 | `Ecp.WorkItemVerifyAlert.List.page` | 工作項逾期警示 |

### 系統管理

| 選單名稱 | 說明 |
|----------|------|
| 系統管理 → 組織架構 → 部門 | 部門架構管理 |
| 系統管理 → 組織架構 → 員工 | 員工資料管理 |
| 系統管理 → 組織架構 → 角色 | 角色管理 |
| 系統管理 → 帳號管理 → 帳號 | 帳號管理 |
| 系統管理 → 帳號管理 → 身份類型 | 身份類型 |

---

## ✅ 新增用戶：推薦方式（API，最快最可靠）

> **優先使用此方式**。不需要操作 UI，直接 API 呼叫，5 步驟完成，實際驗證可用。
> UI 方式見下方 Step 3/4（brittle，僅備用）。

### 完整流程（員工 + 帳號 + 綁定 + 密碼）

```javascript
// 在 browser_run_code 裡一次完成所有步驟
browser_run_code(code="async (page) => {
  const post = (path, body) => page.evaluate(({path, body}) =>
    fetch('https://econtact.ai3.cloud/ecp/' + path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body)
    }).then(r => r.json()), {path, body}
  );

  // Step 1: 建立員工
  const userRes = await post('Qs.User.save.data', {
    data: [{
      FName: '姓名',
      FGender: '1',            // '1'=男 '2'=女
      FDepartmentId: '00000000-0000-0000-1001-000000000001',  // 集團（頂層）
      FDuty: '職務',
      FEmail: 'email@example.com',
      FLanguage: 'zh-tw',
      FEnabled: true
    }]
  });
  const userId = userRes.entityIds[0];

  // Step 2: 建立帳號
  const acctRes = await post('Qs.Account.save.data', {
    data: [{
      FName: '姓名',
      FLoginName: 'loginname',
      FLanguage: 'zh-tw',
      FEnabled: true
    }]
  });
  const accountId = acctRes.entityIds[0];

  // Step 3: 綁定帳號 ↔ 員工
  await post('Qs.AccountIdentity.bind.data', {
    unitId:         '00000000-0000-0000-0001-000000001002',   // 固定值
    accountId:      accountId,
    identityTypeId: '564cf69e-76d6-4baf-b584-6e04c2911dae',  // 固定值（員工身份）
    entityId:       userId
  });

  // Step 4: 設定密碼（不可在 save 時帶 FPassword，否則存成明文無法登入）
  await post('Qs.Account.modifyPassword.data', {
    accountId:          accountId,
    newPassword:        '<EXAMPLE_PASSWORD>',
    requireOldPassword: false
  });

  return { userId, accountId, success: true };
}")
```

### ⚠️ 密碼重要規則
| 方式 | 結果 |
|------|------|
| `Qs.Account.save` 帶 `FPassword` | ❌ 存為**明文**，登入會失敗 |
| `Qs.Account.modifyPassword` + `requireOldPassword:false` | ✅ 正確加密，可登入 |

---

## 🚀 批量建立員工（平行優化）

> **效能提升 17x** — 使用 `Promise.all` 平行執行，20 個員工約 1 秒完成（傳統順序執行需 16 秒）

### 核心優化：Promise.all 平行批次

```javascript
browser_run_code(code="async (page) => {
  // ========== 批次員工建立（平行優化）==========
  
  const startTime = Date.now();
  
  // 員工清單（可替換為任何資料）
  const characters = [
    { name: '于右任', loginName: 'yuyr', duty: '政治家' },
    { name: '戴季陶', loginName: 'daijt', duty: '政治家' },
    { name: '陳果夫', loginName: 'chenggf', duty: '政治家' },
    { name: '何應欽', loginName: 'heyq', duty: '軍事家' },
    { name: '張治中', loginName: 'zhangzz', duty: '軍事家' },
  ];
  
  // 固定參數
  const departmentId = '00000000-0000-0000-1001-000000000001'; // 集團
  const unitId = '00000000-0000-0000-0001-000000001002';
  const identityTypeId = '564cf69e-76d6-4baf-b584-6e04c2911dae';
  
  // API 封裝
  const post = (path, body) => page.evaluate(({path, body}) =>
    fetch('https://econtact.ai3.cloud/ecp/' + path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body)
    }).then(r => r.json()), {path, body}
  );

  // Step 1: 平行建立所有員工
  const t1 = Date.now();
  const userResults = await Promise.all(
    characters.map(char => post('Qs.User.save.data', {
      data: [{
        FName: char.name,
        FGender: '1',
        FDepartmentId: departmentId,
        FDuty: char.duty,
        FLanguage: 'zh-tw',
        FEnabled: true
      }]
    }).then(r => ({ char, result: r })))
  );
  console.log('Step 1 done in ' + (Date.now()-t1) + 'ms');
  
  const users = userResults.map(r => ({
    ...r.char,
    userId: r.result.entityIds?.[0],
    success: !!r.result.entityIds?.[0]
  }));
  
  // Step 2: 平行建立所有帳號
  const t2 = Date.now();
  const accountResults = await Promise.all(
    users.filter(u => u.success).map(user => 
      post('Qs.Account.save.data', {
        data: [{
          FName: user.name,
          FLoginName: user.loginName,
          FLanguage: 'zh-tw',
          FEnabled: true
        }]
      }).then(r => ({ user, result: r }))
    )
  );
  console.log('Step 2 done in ' + (Date.now()-t2) + 'ms');
  
  const accounts = accountResults.map(r => ({
    ...r.user,
    accountId: r.result.entityIds?.[0],
    success: !!r.result.entityIds?.[0]
  }));
  
  // Step 3: 平行綁定所有帳號 ↔ 員工
  const t3 = Date.now();
  await Promise.all(
    accounts.filter(a => a.success).map(account =>
      post('Qs.AccountIdentity.bind.data', {
        unitId: unitId,
        accountId: account.accountId,
        identityTypeId: identityTypeId,
        entityId: account.userId
      })
    )
  );
  console.log('Step 3 done in ' + (Date.now()-t3) + 'ms');
  
  // Step 4: 平行設定所有密碼
  const t4 = Date.now();
  await Promise.all(
    accounts.filter(a => a.success).map(account =>
      post('Qs.Account.modifyPassword.data', {
        accountId: account.accountId,
        newPassword=<PASSWORD_PLACEHOLDER>,
        requireOldPassword: false
      })
    )
  );
  console.log('Step 4 done in ' + (Date.now()-t4) + 'ms');
  
  const elapsed = Date.now() - startTime;
  const successCount = accounts.filter(a => a.success).length;
  
  return {
    total: characters.length,
    success: successCount,
    failed: characters.length - successCount,
    totalElapsed_ms: elapsed,
    accounts: accounts
  };
}")
```

### 效能對比

| 方式 | 5 員工 | 20 員工 |
|------|--------|---------|
| 順序執行（舊） | ~4,000ms | ~16,000ms |
| **平行批次（新）** | **~235ms** | **~940ms** |
| **提升** | **17x** | **17x** |

### 設計原則

| 原則 | 說明 |
|------|------|
| **同 step 平行** | Step 1 所有員工一起建立，Step 2 所有帳號一起建立，以此類推 |
| **錯誤隔離** | 單一失敗不影響其他 |
| **計時追蹤** | 每 step 顯示耗時，方便診斷瓶頸 |
| **結果回傳** | 回傳成功/失敗數、耗時、詳細帳號資料 |

### 固定參數（不需查詢，直接用）
| 參數 | 值 | 說明 |
|------|-----|------|
| `unitId` | `00000000-0000-0000-0001-000000001002` | Qs.User 單元 ID |
| `identityTypeId` | `564cf69e-76d6-4baf-b584-6e04c2911dae` | 員工身份類型 |
| 集團 `FDepartmentId` | `00000000-0000-0000-1001-000000000001` | 頂層部門 |

---

## 🗑️ 批量刪除員工（平行優化）

> 使用 `Promise.all` 平行刪除，效能比順序刪除提升約 17x

### 刪除原則

**重要**：刪除員工前必須先刪除關聯的帳號，否則會報錯：
```text
Qs.Entity.CannotDelete2: 不能刪除員工，因為存在相關聯的帳號身份。
```

### 刪除流程

1. **Step 1**: 平行刪除所有帳號
2. **Step 2**: 平行刪除所有員工

### 完整程式碼

```javascript
browser_run_code(code="async (page) => {
  const post = (path, body) => page.evaluate(({path, body}) =>
    fetch('https://econtact.ai3.cloud/ecp/' + path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body)
    }).then(r => r.json()), {path, body}
  );

  // 要刪除的員工和帳號 ID 清單
  // （從當初建立的結果中取得）
  const employees = [
    { name: '蔣中正', userId: '19d04a7f-be00-03a9-662e-e66a42541f0a', accountId: '19d04a8d-fc20-03a9-662e-e66a42541f0a' },
    { name: '孫中山', userId: '19d04a7f-cd60-03a9-662e-e66a42541f0a', accountId: '19d04a8e-0c90-03a9-662e-e66a42541f0a' },
    // ... 其他員工
  ];

  const accountIds = employees.map(e => e.accountId);
  const userIds = employees.map(e => e.userId);

  // Step 1: 平行刪除所有帳號
  const t1 = Date.now();
  await Promise.all(accountIds.map(id =>
    post('Qs.Account.delete.data', { entityIds: [id] })
  ));
  console.log('帳號刪除完成: ' + (Date.now()-t1) + 'ms');

  // Step 2: 平行刪除所有員工
  const t2 = Date.now();
  await Promise.all(userIds.map(id =>
    post('Qs.User.delete.data', { entityIds: [id] })
  ));
  console.log('員工刪除完成: ' + (Date.now()-t2) + 'ms');

  // 驗證刪除結果
  const verify = await post('Qs.User.getListData.data', {
    pageIndex: 0, pageSize: 50, filtered: true
  });
  const remaining = verify.data?.records?.filter(r => userIds.includes(r.FId)) || [];

  return {
    deletedCount: employees.length,
    remainingCount: remaining.length,
    remainingNames: remaining.map(r => r.FName),
    totalElapsed_ms: Date.now() - t1 - t2
  };
}")
```

### 常見錯誤

| 錯誤訊息 | 原因 | 解法 |
|----------|------|------|
| `Qs.Entity.CannotDelete2` | 帳號未刪除就先刪員工 | 先刪帳號，再刪員工 |
| `{}`（空物件） | 刪除成功（無內容回傳） | 驗證員工列表確認 |

---

## Step 3：新增員工完整流程（UI 備用方式）

> **建議優先使用上方 API 方式。** 此 UI 方式適合需要人工確認欄位的情境。

### 3a. 進入員工列表並點擊新增

```javascript
browser_run_code(code="async (page) => {
  await page.getByText('系統管理').nth(1).click();
  await page.waitForTimeout(300);
  await page.getByText('組織架構').click();
  await page.waitForTimeout(300);
  await page.getByText('員工').click();
  await page.waitForTimeout(800);
  return 'navigated';
}")
// 再 snapshot 確認員工列表載入，找到「新增」按鈕的 ref
browser_snapshot()
browser_click(element="新增", ref="fXXeXX")
```

### 3b. 填寫員工基本資料

**必填欄位：姓名、性別、語言**

```text
// 姓名
browser_type(element="姓名 textbox", ref="fXXe40", text="張飛")

// 性別（下拉選單）
browser_click(element="性別 textbox", ref="fXXe46")
// 等下拉選單出現
browser_click(element="男", ref="eXXX")

// 語言（下拉選單）
browser_click(element="語言 textbox", ref="fXXe121")
// 等下拉選單出現
browser_click(element="繁體中文", ref="eXXX")

// 選填：職務、電子信箱、分機號碼、描述
browser_type(element="職務 textbox", ref="fXXe114", text="蜀漢車騎將軍")
browser_type(element="電子信箱 textbox", ref="fXXe128", text="zhangfei@threekingdoms.com")
browser_type(element="分機號碼 textbox", ref="fXXe134", text="004")
browser_type(element="描述 textbox", ref="fXXe146", text="燕人張飛，蜀漢五虎上將，勇武豪義")
```

### 3c. 保存員工

```text
browser_click(element="保存", ref="fXXe14")
// 確認出現「保存成功」消息
browser_snapshot()
```

---

## Step 4：為員工新增帳號並設定密碼（UI 備用方式）

> **建議優先使用上方 API 方式。**

### 4a. 在員工表單切換到「帳號」標籤

員工保存成功後（或從員工列表打開員工），點擊「帳號」標籤：

```text
browser_click(element="帳號 tab", ref="fXXe24")
```

### 4b. 點擊「新增帳號並關聯」

```text
browser_click(element="新增帳號並關聯", ref="fXXe10")
```

### 4c. 填寫帳號資料

**必填：登入名、語言**

```text
browser_type(element="登入名 textbox", ref="fXXe46", text="zhangfei")

// 語言下拉選單
browser_click(element="語言 textbox", ref="fXXe52")
browser_click(element="繁體中文", ref="eXXX")
```

### 4d. 保存帳號

```text
browser_click(element="保存", ref="fXXe10")
// 確認「保存成功」
```

### 4e. 設定密碼

```text
// 點擊帳號列表中的帳號 row
browser_click(element="張飛 account row", ref="fXXeXX")

// 點擊「設定密碼」
browser_click(element="設定密碼", ref="fXXe22")

// 填寫密碼
browser_type(element="新密碼", ref="fXXe36", text="<EXAMPLE_PASSWORD>")
browser_type(element="確認新密碼", ref="fXXe42", text="<EXAMPLE_PASSWORD>")

// 確定
browser_click(element="確定", ref="fXXe10")
// 確認「密碼修改成功」
browser_snapshot()
// 關閉成功提示
browser_click(element="確定", ref="eXXX")
```

---

## Step 5：關閉對話框

操作完成後關閉表單對話框：

```javascript
browser_run_code(code="async (page) => {
  const closeButtons = await page.locator('.JuiDialogClose').all();
  for (const btn of closeButtons) {
    try {
      if (await btn.isVisible()) {
        await btn.click();
        await page.waitForTimeout(300);
      }
    } catch (e) {}
  }
  return 'closed';
}")
```

---

## 📋 已建立的帳號（上古神話人物）

> 統一密碼：`Pangu123`

| 員工姓名 | 登入名 | 密碼 | 職務 |
|----------|--------|------|------|
| 盤古 | pangu | Pangu123 | 創世之神 |
| 女媧 | nuwa | Pangu123 | 補天之神 |
| 伏羲 | fuxi | Pangu123 | 三皇之一 |
| 神農 | shennong | Pangu123 | 農業醫藥之神 |
| 黃帝 | huangdi | Pangu123 | 華夏始祖 |
| 堯 | yao | Pangu123 | 上古賢帝 |
| 舜 | shun | Pangu123 | 禪讓聖王 |
| 禹 | yu | Pangu123 | 大禹治水 |
| 倉頡 | cangjie | Pangu123 | 漢字發明者 |
| 后羿 | houyi | Pangu123 | 射日英雄 |
| 嫦娥 | change | Pangu123 | 月宮仙子 |
| 夸父 | kuafu | Pangu123 | 逐日英雄 |
| 豬八戒 | zhubajie | Pangu123 | （舊帳號，西遊記） |

### 🔐 修改帳號密碼（UI 流程）

> 適用於修改已存在帳號的密碼（黃建雄密碼修改於 2026-03-19）

**完整流程：**
```javascript
// Step 1: 導航到帳號管理
browser_run_code(code="async (page) => {
  await page.getByText('系統管理').nth(1).click();
  await page.waitForTimeout(300);
  await page.getByText('帳號管理').click();
  await page.waitForTimeout(300);
  await page.getByText('帳號').click();
  await page.waitForTimeout(800);
  return 'navigated';
}")

// Step 2: 找到目標帳號並點擊
browser_snapshot()
// 點擊帳號列
browser_click(element="James account row", ref="fXXeXX")

// Step 3: 點擊「設定密碼」
browser_click(element="設定密碼", ref="fXXe22")

// Step 4: 在修改密碼對話框填入新密碼
browser_fill_form(fields=[
  {"name": "新密碼", "ref": "fXXe36", "type": "textbox", "value": "<EXAMPLE_PASSWORD>"},
  {"name": "確認新密碼", "ref": "fXXe42", "type": "textbox", "value": "<EXAMPLE_PASSWORD>"}
])

// Step 5: 點擊確定
browser_click(element="確定", ref="fXXe10")

// Step 6: 確認成功訊息「密碼修改成功」
browser_snapshot()
// 關閉成功提示
browser_click(element="確定", ref="eXXX")
```

---

### 🔐 修改帳號密碼（API 流程）

> 如果已知帳號 ID，可直接使用 API 修改密碼，效率比 UI 更高

**API 端點：** `POST https://econtact.ai3.cloud/ecp/Qs.Account.modifyPassword.data`

**必要參數：**
| 參數 | 類型 | 說明 |
|------|------|------|
| `accountId` | string | 帳號 ID |
| `newPassword` | string | 新密碼 |
| `requireOldPassword` | boolean | 是否需要舊密碼（通常傳 `false`） |

**curl 範例：**
```bash
# 假設已知帳號 ID為 19d04a8d-fc20-03a9-662e-e66a42541f0a
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.modifyPassword.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{
    "accountId": "19d04a8d-fc20-03a9-662e-e66a42541f0a",
    "newPassword": "<EXAMPLE_PASSWORD>",
    "requireOldPassword": false
  }'
# 成功回傳：{}
```

**JavaScript（在 browser_run_code 中執行）：**
```javascript
browser_run_code(code="async (page) => {
  const result = await page.evaluate(() => {
    return fetch('https://econtact.ai3.cloud/ecp/Qs.Account.modifyPassword.data', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        accountId: '19d04a8d-fc20-03a9-662e-e66a42541f0a',
        newPassword: '<EXAMPLE_PASSWORD>',
        requireOldPassword: false
      })
    }).then(r => r.json());
  });
  return result; // {} = 成功
}")
```

**重要提醒：**
- `requireOldPassword: false` 表示不需要舊密碼即可修改（管理員權限）
- 帳號 ID 可從 `Qs.Account.getListData` API 取得

## 🔑 密碼規則（實際驗證）

| 規則 | 說明 | 範例 |
|------|------|------|
| ✅ 長度 | 最少 8 碼 | `<EXAMPLE_PASSWORD>` |
| ✅ 需混合 | 大寫 + 小寫 + 數字 | `<EXAMPLE_PASSWORD>` ✅ |
| ❌ 同字元連續超過 3 次 | 系統拒絕 | `Chainsea@0000`（0 重複 4 次）❌ |
| ❌ 最近 3 次歷史 | 不能重複使用前 3 次密碼 | 需先改其他密碼再改回來 |

**合規密碼範例：** `<EXAMPLE_PASSWORD>`、`<EXAMPLE_PASSWORD>`、`<ECP_PASSWORD>`

**錯誤碼對照：**
| 錯誤碼 | 原因 | 解法 |
|--------|------|------|
| `{}` | 修改成功 | — |
| `Qs.User.PasswordDuplicateWithHistory` | 與最近 3 次密碼相同 | 先改成其他密碼，再改回目標密碼 |
| `Qs.User.PasswordInvalid` 或類似 | 密碼格式不符規則 | 確認無連續 4 個相同字元、有大小寫+數字 |

### ⚠️ 關於預設角色

**預設角色會自動分配給所有帳號，不需要手動設定。**

若嘗試手動新增預設角色，系統會報錯：
> 「不能為使用者分配預設角色，因為全部使用者已經自動擁有預設角色。」

---

## Step 6：查詢資料（進階）

### 通用：從 iframe 讀取資料

```javascript
browser_run_code(code="async (page) => {
  const iframes = Array.from(document.querySelectorAll('iframe'));
  const targetFrame = iframes.find(f => f.src.includes('CheckIn'));
  if (!targetFrame) return { error: '找不到目標 iframe' };
  const doc = targetFrame.contentDocument || targetFrame.contentWindow.document;
  return { bodyText: doc.body.innerText.substring(0, 3000) };
}")
```

---

## ⚙️ 常見問題排除

| 問題 | 解法 |
|------|------|
| **「要離開網站嗎？」對話框卡住** | ✅ 根本解法：**開新分頁**，用 Step 1d 的 `addInitScript` 攔截。舊分頁無法補救，必須重開。 |
| 導航逾時 / 頁面卡住（非 beforeunload） | `browser_handle_dialog(accept=true)` 處理其他類型對話框 |
| 登入後要求確認重複登入 | `browser_snapshot()` 找到確認按鈕 ref，點擊確定 |
| 選單點擊沒反應 | 改用 `browser_run_code` 的 `page.getByText().click()` 方式 |
| ref 點擊逾時（element is outside of the viewport）| 改用 `browser_run_code` 文字比對方式操作 |
| 對話框遮住畫面（JuiDialogMask / JuiMessageBoxMask 擋住） | 見下方「Mask 遮罩阻擋」章節 |
| **清單頁行選不中**（row click 無反應或被 iframe 擋住）| **原因**：數據列 left=-261 在視窗外。**解法**：用 JS 點擊 `.JuiListLeftTable tr`的 `cells[1]`（check cell）來選中行，不要直接點擊數據列 |
| 下拉選單無法輸入 | 先點擊開啟選單，等待選項出現後再點擊選項 |
| 保存失敗 | 確認必填欄位（姓名、性別、語言）都已填寫 |
| 頁面內容空白 | 可能是 iframe 未載入，等待後再 `browser_snapshot()` |
| 系統日誌顯示 WebSocket 錯誤 | 正常現象，不影響操作 |
| **瀏覽器問「是否儲存密碼」** | 這是 Chrome 原生 UI，無法透過 DOM / Playwright 腳本關閉。需在 MCP server 啟動設定加 `--disable-save-password-bubble` 參數（環境層級設定） |

### Mask 遮罩阻擋：Playwright 點擊 Timeout 的根本解法

**症狀**：
```text
<div class="JuiMessageBoxMask" intercepts pointer events>
<div class="JuiDialogMask" intercepts pointer events>
```

**解法：在每次 JS 點擊前，先移除所有遮罩**

```javascript
browser_evaluate(() => {
  document.querySelectorAll('.JuiMessageBoxMask, .JuiDialogMask, .JuiMask').forEach(m => m.remove());
})
```

然後再用 JS 或 Playwright 點擊目標元素。

**千萬不要**嘗試用 Playwright 的 `force: true` 對著 mask 底下的按鈕點擊——mask 會在視圖中持續存在，擋住所有互動。

---

## 🔐 帳號資訊

- **帳號**：James.Huang
- **密碼**：<ECP_PASSWORD>

---

## ⚠️ Qs.Account.getListData API 的 Scope 限制與分頁方式（2026-04-03 發現並修正）

> **重要發現**：`Qs.Account.getListData` 直接呼叫（`/ecp/Qs.Account.getListData.data`）無論更換 `pageIndex` 都只回傳同一批 25 筆記錄。但 UI 分頁使用的是另一個隱藏端點，可以正確分頁。

### 分頁對比

| 查詢方式 | API 路徑 | 分頁是否正常 | 說明 |
|----------|----------|-------------|------|
| 直接呼叫 `Qs.Account.getListData` | `/ecp/Qs.Account.getListData.data` | ❌ 否 | pageIndex 被忽略，總是返回相同 25 筆 |
| UI 分頁使用的端點 | `/ecp/qsvd-list/Qs.Account.getListData.data` | ✅ 是 | 需要 `listId`，可正確換頁 |

### UI 分頁 API（正確方式）

ECP 網頁切換頁面時的真實請求：

```bash
# 登入後查第 N 頁（必須先有 listId）
curl -s -b cookies.txt -X POST "http://10.145.119.234:12821/ecp/qsvd-list/Qs.Account.getListData.data" \
  -H "Content-Type: application/json" \
  -H "qs-pagecode: Qs.Account.List" \
  -d '{"pageIndex":1,"listId":"16bf9eef-19f0-0574-10c7-acde48001122","keyword":"","isRefresh":false}'
```

**參數說明：**

| 參數 | 說明 |
|------|------|
| `pageIndex` | 頁碼（1-indexed） |
| `listId` | 清單 ID，固定值 `16bf9eef-19f0-0574-10c7-acde48001122` |
| `keyword` | 搜尋關鍵字（空字串表示不篩選） |
| `isRefresh` | `true` = 第一次載入，`false` = 換頁 |

**分頁結果（4 頁，共 80 筆）：**

| 頁 | 內容 |
|----|------|
| 第 1 頁 | api_user + 25 個神話人物（盤古、女媧、伏羲...倉頡、后羿、嫦娥、夸父） |
| 第 2 頁 | 25 個演員明星（林依晨、楊丞琳、陳喬恩...古力娜扎、周迅、章子怡） |
| 第 3 頁 | 25 個演員明星（范冰冰、徐若瑄、蕭亞軒...柯震東） |
| 第 4 頁 | 鄭元暢、張睿家、林柏宏、Administrator（最後一頁，`hasNextPage: false`） |

### Scope 限制仍存在

即使是 `qsvd-list` 分頁 API，**仍然受到 Scope 限制**：
- 看不見：James.Huang
- 可看見：Administrator（但密碼非 <ECP_PASSWORD>）
- SQL 查詢有 79 筆，分頁 API 顯示 80 筆（顯示多了 Administrator 帳號）

### 解決方案

#### 方案 A：使用 `Qs.Misc.executeSql`（完整繞過 Scope）

```bash
# 登入後執行 SQL 直接查 TsAccount 表
curl -s -b cookies.txt -X POST "http://10.145.119.234:12821/ecp/Qs.Misc.executeSql.data" \
  -H "Content-Type: application/json" \
  -d '{"dataSource":"default","sql":"SELECT FId, FLoginName, FName, FEmail, FEnabled, FCreateTime FROM TsAccount ORDER BY FCreateTime DESC LIMIT 200"}'
```

可取得**全部 79 筆**帳號（含 James.Huang）。

#### 方案 B：使用 curl 直接登入 ECP 並用分頁 API

```bash
# Step 1: 登入
curl -s -c cookies.txt -X POST "http://10.145.119.234:12821/ecp/Qs.OnlineUser.login.data" \
  -H "Content-Type: application/json" \
  -d '{"loginName":"Administrator","password":"<ECP_PASSWORD>","language":"zh-tw"}'

# Step 2: 查第 1 頁（分頁 API，有 Scope 限制，只能拿到 26 筆）
curl -s -b cookies.txt -X POST "http://10.145.119.234:12821/ecp/qsvd-list/Qs.Account.getListData.data" \
  -H "Content-Type: application/json" \
  -H "qs-pagecode: Qs.Account.List" \
  -d '{"pageIndex":1,"listId":"16bf9eef-19f0-0574-10c7-acde48001122","keyword":"","isRefresh":true}'
```

### 為何直接呼叫 `Qs.Account.getListData` 的 pageIndex 會被忽略？

這是 ECP Scope 機制的設計 — 後端根據**當前帳號的資料權限**計算總數，而非實際資料庫總數。即使傳入 `pageIndex=1`，後端仍返回第 0 頁的範圍（因為權限只允許看到那 25 筆）。`hasNextPage: true` 是誤導，表示「還有更多」（但沒有查看其餘資料的權限）。

---

## 🛡️ 權限系統：API 不能繞過限制（實際驗證）

> **結論：ECP 權限控制在後端 Java 層，直接打 API 無法繞過。**
> UI 只是「不顯示按鈕」，真正的權限檢查在 ActionImpl 裡。

### 權限測試結果（趙雲帳號，零角色狀態）

| API 操作 | 結果 | 錯誤碼 |
|----------|------|--------|
| `qsvd-list/Qs.User.getListData` — 員工列表 | ✅ 可查詢 | — |
| `qsvd-list/Qs.Role.getListData` — 角色列表 | ✅ 可查詢 | — |
| `Qs.OnlineUser.getCurrentUserInformation` — 自己資訊 | ✅ 永遠可查 | — |
| `Qs.User.save` — 建立員工 | ❌ 被擋 | `Qs.Privilege.CannotCreate` |
| `Qs.User.save` (帶FId) — 修改員工 | ❌ 被擋 | `Qs.Privilege.Check2` |
| `Qs.User.delete` — 刪除員工 | ❌ 被擋 | `Qs.Privilege.Check2` |
| `Qs.Account.modifyPassword` — 改密碼 | ❌ 被擋 | `Qs.Privilege.Check2` |
| `Qs.Menu.setRoleMenus` — 設定角色功能表 | ❌ 被擋 | `Qs.Privilege.Check2` |
| `Qs.Misc.executeSql` — 執行 SQL | ❌ 被擋 | `Qs.Privilege.James.HuangRequired` |
| `Qs.Monitor.clearCache` — 清除快取 | ❌ 被擋 | `Qs.Privilege.Check1` |
| `qsvd-list/Qs.Account.getListData` — 帳號列表 | ✅ 但回傳空 | Scope 過濾 |
| `qsvd-list/Ecp.Contact.getListData` — 聯絡人 | ✅ 但回傳空 | Scope 過濾 |

### 兩層保護機制

```text
UI 層     → 沒有按鈕/選單（只是視覺隱藏，可繞過）
                ↓
API 後端  → Java ActionImpl 權限檢查 → 真正擋住，無法繞過
```

### 無角色帳號的行為規律

- **讀取(getListData)**：能呼叫但資料被 Scope 過濾 — 只看到「自己有查看權限」的資料（通常是空的）
- **寫入(save/delete/modify)**：直接回傳權限錯誤，完全擋住
- **系統管理操作**：`James.HuangRequired`，只有 James.Huang 帳號才能執行
- **自己的資訊**：`getCurrentUserInformation` 永遠可以查

### 常見權限錯誤碼

| 錯誤碼 | 意義 |
|--------|------|
| `Qs.Privilege.CannotCreate` | 無「新增」權限 |
| `Qs.Privilege.Check1` | 無功能表/模組存取權限 |
| `Qs.Privilege.Check2` | 無資料操作權限（修改/刪除/管理） |
| `Qs.Privilege.James.HuangRequired` | 需要系統管理員 |
| `Qs.Auth.SessionInvalid` | Session 過期或未登入 |

---

## 📚 API 原始碼位置及完整單元表格

### JAR 檔案位置（Docker 容器 `ecp-unlock` 內）

| 檔案 | 容器內路徑 | 大小 | 說明 |
|------|-----------|------|------|
| **ECP 業務模組** | `/app/apache-tomcat/webapps/ecp/WEB-INF/lib/ecp-module-main-8.5.03.02.jar` | 43.6 MB | ECP 自訂業務邏輯（API 實作） |
| **ECP 業務模組（副本）** | `/tmp/ecp-module-main-8.5.03.02.jar` | 43.6 MB | 容器 /tmp 目錄下的副本 |
| **Quicksilver 框架** | `/tmp/quicksilver-module-main-7.1.31.beta25.jar` | 16 MB | 平台框架（通用 CRUD 等基底類別） |

> 若要取回原始碼 JAR：<br>
> `docker cp ecp-unlock:/tmp/ecp-module-main-8.5.03.02.jar ./`<br>
> `scp hch@10.145.119.234:/tmp/ecp-module-main-8.5.03.02.jar ./`

### API 類別在 JAR 內的路徑

JAR 內的 class 檔案結構（可直接用 `unzip -l` 查看）：
```text
com/chainsea/ecp/{模組}/api/impl/{Entity}ApiImpl.class      ← API 實作
com/chainsea/ecp/{模組}/action/impl/{Entity}ActionImpl.class  ← Action 實作
com/jeedsoft/quicksilver/base/api/impl/EntityApiImpl.class   ← 框架基底（含 getItem/getList/save/delete）
com/jeedsoft/quicksilver/account/api/impl/AccountApiImpl.class
```

---

### ✅ 完整 API 單元表格（50 個已啟用單元，2026-03-19 實測）

> 符號說明：🟢 = 已驗證可用　⚪ = 未驗證（可能可用）　🔴 = 需特殊權限

#### ECP CRM 模組（25 個）

| UnitCode | 資料表 | 中文名 | getListData | save | getItem | delete | 備註 |
|----------|--------|--------|:-----------:|:----:|:-------:|:------:|------|
| `Ecp.Activity` | TcActivity | 活動 | 🟢 | 🟢 | 🔴 | 🔴 | |
| `Ecp.AiAgentFunction` | TcAiAgentFunction | AI Agent 任務 | 🟢 | 🔴 | 🔴 | 🟢 | |
| `Ecp.AIMessage` | TcAIMessage | AI 對話紀錄 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.AIRoom` | TcAIRoom | 對話服務 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.CheckIn` | — | **出勤打卡** | 🟢 | 🔴 | 🟢 | 🔴 | 僅讀取 |
| `Ecp.ChatAddressBook` | TcChatAddressBook | 聊天通訊錄 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.ChatMessage` | TcChatMessage | 聊天消息 | 🔴 | 🔴 | 🔴 | 🔴 | 需其他 API 上下文 |
| `Ecp.ChatMonitor` | — | 聊天監控 | 🔴 | 🔴 | 🔴 | 🔴 | |
| `Ecp.ChatRoom` | — | 聊天室 | 🔴 | 🔴 | 🔴 | 🔴 | |
| `Ecp.ChatWorkGroup` | TcChatWorkGroup | 文字客服群組 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.Contact` | TcContact | **聯絡人** | 🟢 | 🟢 | 🟢 | 🟢 | **最常用** |
| `Ecp.ContactLabel` | TcContactLabel | 標籤變數 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.CopilotGreeting` | TcCopilotGreeting | Copilot 招呼語 | 🟢 | 🔴 | 🔴 | 🔴 | |
| `Ecp.CopilotMessage` | TcCopilotMessage | Copilot 對話紀錄 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.CopilotVerifyList` | TcCopilotVerifyList | 驗證任務 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.CrawlerSchedule` | TcCrawlerSchedule | 社群排程 | 🟢 | 🔴 | 🔴 | 🟢 | |
| `Ecp.Customer` | TcCustomer | **企業客戶** | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.ExploreMap` | TcExploreMap | 探索地圖 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.ExpressChat` | — | ExpressChat | 🔴 | 🔴 | 🔴 | 🔴 | |
| `Ecp.ExpressChatMaster` | TcExpressChatMaster | 服務總機管理 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.ExpressChatParameter` | TcExpressChatParameter | 參數文案管理 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.FbFans` | TcFbFans | 社群媒體 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.Lead` | TcLead | **線索** | 🟢 | 🟢 | 🔴 | 🔴 | |
| `Ecp.MarketContent` | TcMarketContent | 行銷文案 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.OfflineCenter` | TcOfflineCenter | 訊息中心 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.OfflineReply` | TcOfflineReply | 訊息回復 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.OfflineRoom` | TcOfflineRoom | 訊息聊天室 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.Opportunity` | TcOpportunity | **商機** | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.Project` | TcProject | **專案** | 🟢 | 🔴 | 🔴 | 🟢 | |
| `Ecp.ProlongTask` | TcProlongTask | 延長任務 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.Prompt` | TcPrompt | AI 提詞器 | 🟢 | 🔴 | 🔴 | 🟢 | |
| `Ecp.Schedule` | TcSchedule | **日程** | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.ServiceRequest` | TcServiceRequest | **服務請求** | 🟢 | 🟢 | 🟢 | 🔴 | |
| `Ecp.ServiceRequestDelay` | TcServiceRequestDelay | 延時申請-服務請求 | 🟢 | 🔴 | 🔴 | 🟢 | |
| `Ecp.SmartQA` | TcSmartQA | 智能問答 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.Task` | TcTask | **任務** | 🟢 | 🟢 | 🔴 | 🔴 | |
| `Ecp.TimeReport` | TcTimeReport | **工時日誌** | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Ecp.TimeReportDetail` | TcTimeReportDetail | 工時日誌明細 | 🟢 | 🟢 | 🔴 | 🟢 | |

#### Quicksilver 平台模組（13 個）

| UnitCode | 資料表 | 中文名 | getListData | save | getItem | delete | 備註 |
|----------|--------|--------|:-----------:|:----:|:-------:|:------:|------|
| `Qs.Account` | TsAccount | **帳號** | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Qs.AccountIdentity` | TsAccountIdentity | 帳號身份 | 🔴 | 🟢 | 🔴 | 🔴 | |
| `Qs.Attachment` | TsAttachment | 附件 | 🔴 | 🟢 | 🔴 | 🔴 | |
| `Qs.Department` | TsDepartment | **部門** | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Qs.Deputy` | TwDeputy | 代理人 | 🔴 | 🟢 | 🔴 | 🔴 | |
| `Qs.DictionaryItem` | TsDictionaryItem | 字典項 | 🟢 | 🔴 | 🔴 | 🟢 | |
| `Qs.DiskFile` | TsDiskFile | 磁片檔 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Qs.IdentityType` | TsIdentityType | 身份類型 | 🟢 | 🟢 | 🔴 | 🟢 | |
| `Qs.Misc` | — | 平台雜項功能 | 🔴 | 🔴 | 🔴 | 🔴 | executeSql 需 James.Huang |
| `Qs.Picture` | TsPicture | 圖片 | 🟢 | 🟢 | 🔴 | 🔴 | |
| `Qs.QuerySchema` | TsQuerySchema | 查詢方案 | 🟢 | 🔴 | 🔴 | 🔴 | |
| `Qs.Token` | — | Token | 🔴 | 🔴 | 🔴 | 🔴 | |
| `Qs.User` | TsUser | **員工** | 🟢 | 🟢 | 🔴 | 🟢 | |

#### Workflow 工作流模組（1 個）

| UnitCode | 資料表 | 中文名 | getListData | save | getItem | delete | 備註 |
|----------|--------|--------|:-----------:|:----:|:-------:|:------:|------|
| `Wf.Process` | TwProcess | 流程實例 | 🟢 | 🟢 | 🔴 | 🟢 | |

---

### 🔑 標準 API 方法呼叫格式

#### 通用列表查詢 `getListData`
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/{UnitCode}.getListData.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data":{"pageIndex":0,"pageSize":20}}'
```

#### 通用新增/修改 `save`
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/{UnitCode}.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data":[{"FName":"名稱","F欄位":"值"}]}'
```

#### 通用單筆查詢 `getItem`
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/{UnitCode}.getItem.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"entityId":"<ID>"}'
```

#### 通用刪除 `delete`
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/{UnitCode}.delete.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"entityIds":["<ID>"]}'
```

### ⚠️ 常見 API 錯誤原因

| 錯誤訊息 | 原因 | 解法 |
|---------|------|------|
| `Basic.Reflect.NoMethodWithArgument` | 方法不存在或參數格式錯誤 | 確認 URL 中無 `.data` 寫成 `.data.data` |
| `Basic.Json.ObjectPropertyMissing` | 缺少必填欄位（如 save 缺 `data`）| 確認 JSON body 包含正確欄位 |
| `Basic.Json.ObjectPropertyTypeCast` | 欄位類型錯誤（如 entityId 傳成字串）| 確認 UUID 格式正確 |
| `Qs.Auth.SessionInvalid` | Session 過期 | 重新登入取得新 cookie |
| `Qs.Privilege.Check2` | 無資料操作權限 | 使用 James.Huang 帳號 |

---

## 📡 API 參考手冊

> 基礎 URL: `https://econtact.ai3.cloud/ecp/`

### 登入方式（必讀）

```bash
# Step 1: 先登入取得 session
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.login.data" \
  -c cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"loginName":"James.Huang","password":"<ECP_PASSWORD>","language":"zh-tw"}'

# Step 2: 後續 API 帶 -b cookies.txt
```

> ⚠️ **API 手冊注意**：下方部分 curl 範例使用舊格式（`saveData`、`deleteData`、`getFormData` 帶 `id` 參數），這些是**未驗證的舊格式**。
> **已驗證的正確格式**請見下方「Write API 正確語法」章節。
> 規律：新增/修改用 `{Unit}.save`（`data` 為 array）；刪除用 `{Unit}.delete`（`entityIds` array）；查詢單筆用 `getFormData`（`entityId` 非 `id`）。

## Quicksilver Framework (Qs.*)

### 帳號/用戶

```bash
# 取得當前用戶資訊 ⚠️ 必須帶 type: 'detail'
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.getCurrentUserInformation.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"type": "detail"}'

# 登入
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.login.data" \
  -c cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"loginName":"James.Huang","password":"<ECP_PASSWORD>","language":"zh-tw"}'

# 登出（會真的登出）
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.logout.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 踢出其他 session ⚠️ 參數是 entityIds（array），不是 sessionId
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.kickOut.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityIds":["xxx"]}'

# 切換身份
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.switchIdentity.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"identityId":"xxx"}'

# 線上用戶列表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 員工列表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20,"filtered":[]}'

# 員工詳細 ⚠️ 參數是 entityId，不是 id
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.getFormData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 新增/修改員工 ⚠️ 正確格式是 save + data array
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.save.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"data":[{'新增欄位...}]}'

# 刪除員工
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.deleteData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityIds":["xxx"]}'

# 啟用/停用員工
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.setEnabled.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityIds":["xxx"],"enabled":true}'

# 取得用戶資訊
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 修改密碼
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.modifyPassword.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"accountId":"xxx","oldPassword":"xxx","newPassword":"xxx"}'

# 啟用/停用帳號
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.setEnabled.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityIds":["xxx"],"enabled":true}'

# 帳號列表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 帳號詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.getFormData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"id":"xxx"}'

# 新增/修改帳號
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.saveData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityDataJson":"{}"}'

# 綁定身份
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.AccountIdentity.bind.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 部門

```bash
# 部門列表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 部門樹狀結構
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.getTreeData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 部門詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.getFormData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"id":"xxx"}'

# 新增/修改部門
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.saveData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityDataJson":"{}"}'

# 刪除部門
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.deleteData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityIds":["xxx"]}'

# 取得部門
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 啟用/停用部門
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.setEnabled.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 角色

```bash
# 角色列表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Role.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 角色詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Role.getFormData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"id":"xxx"}'

# 新增/修改角色
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Role.saveData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityDataJson":"{}"}'

# 刪除角色
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Role.deleteData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityIds":["xxx"]}'

# 儲存角色權限設定
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.RolePrivilege.saveConfig.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 功能表/權限

```bash
# 取得功能表樹
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Menu.getTree.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"parentId":""}'

# 設定角色功能表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Menu.setRoleMenus.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"roleId":"xxx","menuIds":[]}'

# 設定工作檯功能表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Menu.setWorkbenchMenus.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 儲存資料權限
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Privilege.saveDataPrivileges.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"roleId":"xxx","privileges":[]}'

# 刪除資料權限
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Privilege.deleteDataPrivileges.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 儲存角色單元權限
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Privilege.saveRoleUnitPrivilege.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 儲存子單元權限
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Privilege.saveSlaveUnitPrivilege.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 設定用戶可存取欄位
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Privilege.setUserAccessFields.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得可設定權限的欄位
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Privilege.getPrivilegableFields.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 系統工具

```bash
# 執行 SQL
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.executeSql.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"dataSource":"xxx","sql":"SELECT * FROM table"}'

# 取得 SQL 結果列表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.getSqlResultList.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得實體名稱
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.getEntityName.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"unitCode":"xxx","entityId":"xxx"}'

# 取得欄位加密金鑰
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.getFieldEncryptKey.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得欄位解密金鑰
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.getFieldDecryptKey.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得遮罩欄位值
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.getMaskFieldValue.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 儲存長 URL 參數
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.putLongUrlArguments.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 是否開放查看頁面
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.isOpenViewPage.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 儲存 CORS 設定
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.saveCorsConfig.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 寫入日誌
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.WriteLog.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 驗證驗證碼
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Misc.verifyCaptcha.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 清除快取
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Monitor.clearCache.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得快取列表
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Monitor.getCacheListDataJson.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得新訊息數量
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.SystemMessage.getNewItemCount.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得待辦事項
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.SystemMessage.getTodoItems.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 標記已讀
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.SystemMessage.read.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 全部標記已讀
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.SystemMessage.readAll.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得跑馬燈內容
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Marquee.getActiveItems.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 發布跑馬燈
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Marquee.publish.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 計時器

```bash
# 啟用計時器
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Timer.enable.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"timerId":"xxx"}'

# 停用計時器
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Timer.disable.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"timerId":"xxx"}'

# 手動執行計時器
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Timer.runManually.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"timerId":"xxx"}'
```

### 工作流 (Wf.*)

```bash
# 首頁待辦工作項
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.WorkItem.getHomepageTodoListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 完成工作項
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.WorkItem.finish.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"workItemId":"xxx"}'

# 批次通過
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.WorkItem.BatchPass.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"workItemIds":["xxx"]}'

# 轉交工作項
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.WorkItem.transfer.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"workItemId":"xxx","toUserId":"xxx"}'

# 複製工作項
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.WorkItem.clone.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 領取工作項
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.WorkItem.draw.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得處理資訊
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.WorkItem.getHandleInformation.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"workItemId":"xxx"}'

# 建立並啟動流程
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.Process.createAndStart.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 終止流程
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.Process.terminate.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 暫停流程
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.Process.suspend.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 恢復流程
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.Process.resume.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得實體流程列表
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.Process.getEntityProcessListJson.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 啟用工作流
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.Workflow.enable.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 停用工作流
curl -X POST "https://econtact.ai3.cloud/ecp/Wf.Workflow.disable.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

## ECP CRM APIs (Ecp.*)

### 聯絡人

```bash
# 聯絡人列表
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 聯絡人詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 發送簡訊
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.doSendSMS.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 發送通知
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.doSendNotification.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 合併重複聯絡人
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.doMergeDuplicateContacts.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得電話
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.getPhone.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 檢查電話是否存在
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.isPhoneExist.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 更新電話
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.updatePhone.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 刪除匿名
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.deleteAnonymous.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"dayBefore":30}'
```

### 企業

```bash
# 企業詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Customer.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 合併企業
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Customer.merge.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得企業聯絡人
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Customer.getCustomerContact.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得集團圖表資料
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Customer.getGroupGraphData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 服務請求

```bash
# 服務請求詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx"}'

# 分配服務請求
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.doAssign.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"xxx","userId":"xxx"}'

# 執行服務請求
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.doExecute.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 發送服務請求
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.doSend.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 再次處理
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.doAgain.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取消關閉
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.doUnClose.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 提交審核
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.submit.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 採購審核
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.purchaseAudit.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 按聯絡人查詢服務請求
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.queryServiceRequestByContact.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 發送簡訊
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.doSendSMS.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 檢查權限
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ServiceRequest.doCheckPrivilege.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 聊天/客服

```bash
# 客服群組登入狀態
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatWorkGroup.getLoginListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得群組資訊
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatWorkGroup.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"workGroupId":"xxx"}'

# 取得客服服務房間
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.getAgentServiceRoom.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得新聊天室
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.getNewItems.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得聊天參數
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.getChatParameters.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 建立聊天室
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.create.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"userIds":["xxx"]}'

# 停止聊天
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.chatStop.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"roomId":"xxx"}'

# 轉交給客服
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.transferToAgent.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 轉交給群組
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.transferToGroup.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 接聽
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.pickUp.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 發起滿意度調查
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.agentSurvey.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 更新聯絡人
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.updateContact.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"roomId":"xxx","contactId":"xxx"}'

# 更新最後閱讀時間
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.updateLastReadTime.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"roomId":"xxx"}'

# 取得聊天室資訊
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatRoom.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"roomId":"xxx"}'

# 發送訊息
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatMessage.send.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得訊息歷史
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatMessage.getHistory.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 是否有新訊息
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatMessage.hasNew.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得表情符號
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatMessage.getEmoji.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 客服就緒
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatAgent.ready.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 客服未就緒
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatAgent.unready.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得工具列狀態
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatAgent.getToolBarStatus.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 檢查是否客服角色
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatAgent.checkAgentIsCustomerServiceRole.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得監控資料
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatMonitor.getMonitorData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得 ASD 資料
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatMonitor.getAsdData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得督導參數
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatSupervisor.getEcpChatSupervisorParameters.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得未就緒狀態
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ChatSupervisor.getUnreadyStatus.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 活動/任務

```bash
# 活動詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Activity.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 完成活動
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Activity.finish.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 首頁行事曆資料
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Activity.getHomepageCalendarListDataJson.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 任務詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Task.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 提交任務
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Task.submit.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 完成確認
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Task.doFinishCheck.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取消關閉
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Task.doUnClose.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 出勤打卡 `Ecp.CheckIn`

```bash
# 查詢出勤列表
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.CheckIn.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"data":{"pageIndex":0,"pageSize":50}}'

# 查詢單筆出勤
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.CheckIn.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId":"<出勤記錄ID>"}'
```

**欄位說明：**

| 欄位 | 說明 |
|------|------|
| `FId` | 出勤記錄 ID |
| `FUserId` | 員工 ID |
| `FUserId$` | 員工姓名 |
| `FLoginName` | 登入帳號 |
| `FDepartmentId$` | 部門名稱 |
| `FCheckinType` | 打卡類型：`1`=上班 `2`=下班 `5`=預補卡 |
| `F_CheckIncategory$` | 上班 / 下班 |
| `FPreOrReCheckInDate` | 打卡時間（完整）|
| `FPreOrReCheckInTime` | 打卡時間（僅時間）|
| `FRegTime` | 實際註冊時間 |
| `FStatus` | `Complete`=完成 `Audited`=審核通過 |
| `FStatus$` | 完成 / 審核通過 |
| `FMemo` | 備註（預補卡原因）|

> ⚠️ 新增（save）/ 刪除（delete）需要特殊權限，一般用戶僅可查詢。

### 知識庫

```bash
# 知識詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Knowledge.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得知識列表
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Knowledge.getKnowledgeItems.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 知識分類詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.KnowledgeCatalog.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 開始訓練
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.SmartQA.doStartTraining.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得訓練進度
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.SmartQA.getTrainingProgress.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 商機/銷售

```bash
# 標記成功
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Opportunity.success.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 暫停商機
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Opportunity.suspend.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 恢復商機
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Opportunity.resume.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 批次更新
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Opportunity.doBatchUpdate.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 提交線索
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Lead.submit.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 行銷

```bash
# 分配目標客戶
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.TargetCustomer.doAssign.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 派發
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.TargetCustomer.doDispatch.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 完成
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.TargetCustomer.doFinish.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 關閉
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.TargetCustomer.doClose.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 標記失敗
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.TargetCustomer.doFail.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 目標客戶列表
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.TargetCustomer.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 執行問卷
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Survey.doSurvey.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 變更問卷狀態
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Survey.changeStatus.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 通話紀錄

```bash
# 建立通話紀錄
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.CallLog.doCreateCallLog.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得 CTI 參數
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.CallLog.getCtiParameters.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得分機長度
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.CallLog.getExtensionLength.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 更新通話資料
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.CallLog.updateCallData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取得服務軌跡
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.CallLog.getServiceTrack.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 專案

```bash
# 專案詳細
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Project.getItem.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 專案列表
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Project.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 提交專案
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Project.submit.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 更新進度
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Project.updateProgressRate.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 文件

```bash
# 傳送文件
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Document.doTransfer.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 儲存為實體文件
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Document.saveAsEntityDocument.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 發送郵件
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Email.send.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"email":{}}'

# 儲存草稿
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Email.saveAsDraft.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"emailId":"xxx"}'
```

### 合約/發票

```bash
# 簽署合約
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contract.doSign.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 完成合約
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contract.doFinish.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 取消合約
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contract.doCancel.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 修訂合約
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contract.doRevise.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'

# 審核發票
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Invoice.audit.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

## ⚠️ Write API 正確語法（實際驗證）

### 關鍵規律：`save` vs `saveData`

ECP 的新增/修改 API **不是** `saveData`，而是 `{UnitCode}.save`，且 `data` 必須是 **array**。

| 錯誤寫法 | 正確寫法 |
|----------|----------|
| `Qs.User.saveData` | `Qs.User.save` |
| `{ data: {...} }` (object) | `{ data: [{...}] }` (**array**) |
| `{ entityDataJson: "..." }` | `{ data: [{ 欄位: 值 }] }` |

成功回應格式：`{"entityIds": ["新建立的ID"]}`

---

### 建立新員工 `Qs.User.save` ✅ 已驗證

```bash
# Step 1: 登入
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.login.data" \
  -c cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"loginName":"James.Huang","password":"<ECP_PASSWORD>","language":"zh-tw"}'

# Step 2: 建立新員工 (data 必須是 array)
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.save.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{
    "data": [{
      "FName": "趙雲",
      "FGender": "1",
      "FDepartmentId": "00000000-0000-0000-1001-000000000001",
      "FDuty": "蜀漢五虎上將",
      "FEmail": "zhaoyu@threekingdoms.com",
      "FDescription": "常山趙子龍，蜀漢五虎上將，忠勇無雙",
      "FEnabled": true,
      "FLanguage": "zh-tw"
    }]
  }'
# 成功回應: {"entityIds":["19d004ea-..."]}
```

**欄位說明：**

| 欄位 | 必填 | 說明 | 範例值 |
|------|------|------|--------|
| `FName` | ✅ | 姓名 | `"趙雲"` |
| `FGender` | ✅ | 性別 `"1"`=男 `"2"`=女 | `"1"` |
| `FLanguage` | ✅ | 語言代碼 | `"zh-tw"` |
| `FDepartmentId` | ✅ | 部門 ID | `"00000000-0000-0000-1001-000000000001"` (集團) |
| `FEnabled` | ✅ | 啟用 | `true` |
| `FDuty` | ❌ | 職務 | `"蜀漢五虎上將"` |
| `FEmail` | ❌ | 電子信箱 | `"zhaoyu@threekingdoms.com"` |
| `FDescription` | ❌ | 描述 | `"常山趙子龍"` |
| `FExtension` | ❌ | 分機號碼 | `"005"` |
| `FIsSales` | ❌ | 是否銷售人員 | `false` |
| `FIsServices` | ❌ | 是否服務人員 | `false` |

---

## 取得員工詳細資料 `Qs.User.getFormData` ✅

```bash
# getFormData 的 key 是 entityId（不是 id）
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.getFormData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"entityId": "19d004ea-98a0-0ee1-49e1-2ae4694a8577"}'

# 回應: {"editDataJson": {"FId":"...","FName":"趙雲","FGender":"1","FLanguage":"zh-tw",...}}
```

---

## 查詢員工列表 `Qs.User.getListData` ✅

```bash
# 注意：列表查詢不需要前綴
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.getListData.data" \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"data":{"pageIndex":0,"pageSize":20,"filtered":true}}'
```

---

## API 前綴規律總結 ✅ 已驗證

> ⚠️ 測試發現 `qsvd-list/` 前綴**並非必要**。以下為正確格式：

| 操作類型 | 前綴 | 範例 |
|----------|------|------|
| **所有 CRUD 操作** | 無前綴 | `Qs.User.getListData.data` |
| 部門樹 | 無前綴 | `Qs.Department.getTreeData.data` |
| 線上用戶 | 無前綴 | `Qs.OnlineUser.getCurrentUserInformation.data` |
| SQL 執行 | 無前綴 | `Qs.Misc.executeSql.data` |

> `qsvd-list/` 前綴可能用於某些特殊頁面場景，但不影響 API 呼叫。

---

## 🆕 API 參數勘誤（2026-04-01 實測）

| API | 文件錯誤 | 正確參數 |
|-----|---------|---------|
| `Qs.OnlineUser.getCurrentUserInformation` | `{}` | `{"type": "detail"}` |
| `Qs.User.getItem` | `{"id": "xxx"}` | `{"entityId": "xxx"}` |
| `Qs.User.delete` | `{"entityIds": ["xxx"]}` | ✅ 正確 |
| `Qs.OnlineUser.kickOut` | `{"sessionId": "xxx"}` | `{"entityIds": ["xxx"]}` |

---

## 🆕 James.Huang 帳號 Scope 限制（2026-04-01 實測）

James.Huang（技術主任）登入後，**讀取受限**，**寫入幾乎全被擋**：

| API | 結果 | 說明 |
|-----|------|------|
| `Qs.User.save` | ❌ `CannotCreate` | 無新增員工權限 |
| `Qs.User.delete` | ❌ `CannotDelete` | 無刪除員工權限 |
| `Qs.Account.getListData` | ⚠️ 空列表 | Scope 過濾 |
| `Qs.IdentityType.getListData` | ⚠️ 空列表 | Scope 過濾 |
| `Ecp.ServiceRequest.getListData` | ⚠️ 空列表 | Scope 限制 |
| `Ecp.Activity.getListData` | ⚠️ 空列表 | 無資料或 Scope |
| `Ecp.ChatWorkGroup.getListData` | ⚠️ 空列表 | 無資料或 Scope |
| `Ecp.Schedule.getListData` | ⚠️ 空列表 | 無資料或 Scope |

**可正常讀取的模組：**

| API | 結果 | 說明 |
|-----|------|------|
| `Qs.User.getListData` | ✅ 50+ 筆 | 員工列表 |
| `Qs.Department.getListData` | ✅ 22 筆 | 部門列表 |
| `Qs.Department.getTreeData` | ✅ | 部門樹 |
| `Ecp.Contact.getListData` | ✅ 50+ 筆 | 聯絡人 |
| `Ecp.Customer.getListData` | ✅ 50+ 筆 | 企業客戶 |
| `Ecp.Lead.getListData` | ✅ 1 筆 | 線索 |
| `Ecp.Opportunity.getListData` | ✅ 2 筆 | 商機 |
| `Ecp.Task.getListData` | ✅ 9 筆 | 任務 |
| `Ecp.Project.getListData` | ✅ 1 筆 | 專案 |
| `Ecp.CheckIn.getListData` | ✅ 50+ 筆 | 出勤打卡 |
| `Ecp.TimeReport.getListData` | ✅ 50+ 筆 | 工時日誌 |
| `Ecp.Document.getListData` | ✅ 50+ 筆 | 文件 |
| `Ecp.Knowledge.getListData` | ✅ 50+ 筆 | 知識庫 |
| `Wf.WorkItem.getListData` | ✅ 32 筆 | 工作項 |
| `Wf.Process.getListData` | ✅ 50+ 筆 | 流程實例 |

---

## 已驗證可建立的其他資源

依照同樣 `{UnitCode}.save` + `data: [array]` 模式，可建立：

```bash
# 建立部門
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Department.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data": [{"FName": "新部門", "FParentId": "00000000-0000-0000-1001-000000000001", "FEnabled": true}]}'

# 建立角色
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Role.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data": [{"FName": "新角色", "FDepartmentType": "User", "FEnabled": true}]}'
```

---

## 🗂️ 左側選單管理（tsmenu）

> ECP 左側所有頁簽和功能表都儲存在 MariaDB 的 `tsmenu` 資料表，可直接 SQL 新增/刪除。

### 重要規則

| 規則 | 說明 |
|------|------|
| `FId` 必須是 UUID | ❌ 不能用 `test-tab-001`，✅ 要用 `a0000001-0001-0001-0001-00155dae810c` |
| `FPageId` 必須是 UUID | ❌ 不能用 `Qs.User.List.page`，✅ 要從 `tsmenu` 查出對應 UUID |
| 修改後需重啟容器 | `clearCache` 只清查詢快取，選單結構由 JVM 啟動時載入，必須 `docker restart` |

### 常用 FPageId UUID 速查

| 功能 | FPageId UUID |
|------|-------------|
| 員工列表 | `eaa06693-4d59-4d51-905c-1d845ca8903c` |
| 部門管理 | `228470b6-62fc-4ebc-a630-1380cb1043a4` |
| 帳號列表 | `16bf9eef-1490-0574-10c7-acde48001122` |
| 線上使用者 | `8169c956-d6d0-4b72-b728-c554a74b9914` |
| 快取監控 | `cc8587aa-4466-4f8d-a73b-2abf75b95dcb` |

### 新增選單完整流程

```javascript
// Step 1: 用 Qs.Misc.executeSql.data 插入記錄（需 James.Huang 登入）
browser_run_code(code="async (page) => {
  const post = (path, body) => page.evaluate(({path, body}) =>
    fetch('https://econtact.ai3.cloud/ecp/' + path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body)
    }).then(r => r.json()), {path, body}
  );

  // 新增頂層選單（FType: Directory, FParentId: NULL, FTreeLevel: 1）
  const r1 = await post('Qs.Misc.executeSql.data', {
    sql: \"INSERT INTO tsmenu (FId, FParentId, FIndex, FTreeLevel, FTreeSerial, FName, FIcon, FType, FEnabled, FBuiltin, FIsSystemLevel, FAlwaysOpenNewTab, FHideInMainMenu, FSubMenuSource) VALUES ('a0000001-0001-0001-0001-00155dae810c', NULL, 99, 1, '999', '我的選單', 'quicksilver/image/16/Admin.gif', 'Directory', 1, 0, 0, 1, 0, 'MenuTable')\"
  });

  // 新增子頁面（FType: InternalPage，FPageId 必須是 UUID）
  const r2 = await post('Qs.Misc.executeSql.data', {
    sql: \"INSERT INTO tsmenu (FId, FParentId, FIndex, FTreeLevel, FTreeSerial, FName, FIcon, FType, FEnabled, FBuiltin, FIsSystemLevel, FAlwaysOpenNewTab, FHideInMainMenu, FSubMenuSource, FPageId) VALUES ('a0000002-0001-0001-0001-00155dae810c', 'a0000001-0001-0001-0001-00155dae810c', 1, 2, '999.001', '員工列表', 'quicksilver/image/16/AllTask.gif', 'InternalPage', 1, 0, 0, 1, 0, 'MenuTable', 'eaa06693-4d59-4d51-905c-1d845ca8903c')\"
  });

  return { r1: r1.logs?.[0]?.status, r2: r2.logs?.[0]?.status };
  // 成功: { r1: 'success', r2: 'success' }
}")
```

```bash
# Step 2: SSH 進伺服器重啟容器（必要，選單才會生效）
ssh hch@10.145.119.234 "docker restart ecp-unlock"
# 等約 35 秒後再登入瀏覽器即可看到新選單
```

### MariaDB 直接操作（緊急用）

```bash
# 容器名稱: ecp-mariadb
# 密碼: <DB_PASSWORD>，用戶: root，資料庫: ecp
ssh hch@10.145.119.234 "docker exec ecp-mariadb env MYSQL_PWD=<DB_PASSWORD> mysql -u root ecp -e 'SELECT FName, FId FROM tsmenu WHERE FParentId IS NULL ORDER BY FIndex'"

# 查某功能的 FPageId UUID
ssh hch@10.145.119.234 "docker exec ecp-mariadb env MYSQL_PWD=<DB_PASSWORD> mysql -u root ecp -e \"SELECT FName, FPageId FROM tsmenu WHERE FName IN ('員工','帳號','部門','快取監控') AND FPageId IS NOT NULL\""
```

### tsmenu 欄位說明

| 欄位 | 說明 | 範例 |
|------|------|------|
| `FId` | UUID 主鍵 | `a0000001-0001-0001-0001-00155dae810c` |
| `FParentId` | 父節點 UUID（頂層為 NULL） | `NULL` |
| `FIndex` | 排序（數字越大越後面） | `99` |
| `FTreeLevel` | 層級（1=頂層, 2=第二層...） | `1` |
| `FTreeSerial` | 層級序列（格式如 `007.001.002`） | `'999.001'` |
| `FName` | 顯示名稱 | `'員工列表'` |
| `FIcon` | 圖示路徑 | `'quicksilver/image/16/AllTask.gif'` |
| `FType` | `Directory`（目錄）或 `InternalPage`（頁面） | `'InternalPage'` |
| `FPageId` | 頁面 UUID（InternalPage 才需要）| `'eaa06693-...'` |
| `FEnabled` | 是否啟用 | `1` |
| `FAlwaysOpenNewTab` | 是否每次開新頁簽 | `1` |
| `FSubMenuSource` | 固定填 `'MenuTable'` | `'MenuTable'` |

---

## 👥 聯絡人 API（Ecp.Contact）✅ 已驗證

### 新增聯絡人

```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data":[{
    "FName": "姓名",
    "FFirstName": "名", "FLastName": "姓",
    "FGender": "1",
    "FMobile": "0912345678",
    "FPhone": "021234567",
    "FBusinessEmail": "email@example.com",
    "FBusinessAddress": "地址",
    "FDuty": "職稱",
    "FCompanyName": "企業名稱",
    "FBloodType": "O",
    "FBirthday": "2026-01-01",
    "FHobby": "個人愛好",
    "FLanguage": "中文",
    "FMarriage": "0",
    "FNote": "右側閱覽窗格備註",
    "FIsPersonalCustomer": true,
    "FStatus": "1"
  }]}'
# → {"entityIds":["<聯絡人ID>"]}
```

### 查詢列表 / 單筆詳細

```bash
# 列表（無需 qsvd-list 前綴）
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.getListData.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"pageIndex":0,"pageSize":20}'

# 單筆詳細（需先修復 getItem bug，見下方章節）
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.getItem.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"entityId":"<聯絡人ID>"}'
```

### 修改聯絡人欄位（帶 FId）

```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data":[{"FId":"<聯絡人ID>","FNote":"修改後的備註"}]}'
```

### 刪除聯絡人

```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.Contact.delete.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"entityIds":["<聯絡人ID>"]}'
```

### 新增備註記錄（TsNote，出現在「備註」頁籤）

```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Ecp.ContactNoteCollection.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data":[{"FContactId":"<聯絡人ID>","FContent":"備註內容"}]}'
```

### 聯絡人欄位速查

| 欄位 | 必填 | 說明 |
|------|------|------|
| `FName` | ✅ | 姓名 |
| `FGender` | ❌ | 性別 `"1"`=男 `"2"`=女 |
| `FMobile` | ❌ | 手機 |
| `FPhone` | ❌ | 商務電話 |
| `FBusinessEmail` | ❌ | 商務電子信箱 |
| `FBusinessAddress` | ❌ | 商務地址 |
| `FDuty` | ❌ | 職稱 |
| `FCompanyName` | ❌ | 企業名稱 |
| `FBloodType` | ❌ | 血型（A/B/O/AB） |
| `FBirthday` | ❌ | 生日（YYYY-MM-DD） |
| `FMarriage` | ❌ | `"0"`=未婚 `"1"`=已婚 |
| `FNote` | ❌ | **右側閱覽窗格**顯示的備註（單一文字欄位） |
| `FIsPersonalCustomer` | ❌ | 是否個人客戶 `true/false` |

### ⚠️ 備註兩種機制（常見混淆）

| 位置 | 對應 | API |
|------|------|-----|
| 右側閱覽窗格「聯絡人備註」| `TcContact.FNote` 欄位 | `Ecp.Contact.save` 帶 `FNote` |
| 聯絡人表單「備註」頁籤 | `TsNote` 表（多筆） | `Ecp.ContactNoteCollection.save` |

---

## 🔧 Ecp.Contact.getItem Bug 修復記錄

> 系統原本呼叫 `Ecp.Contact.getItem` 會報 SQL 錯誤，已修復。

### 問題根源

`TsField`（欄位設定表）中有 9 個欄位設定為 `FSourceType = 'local'`，但這些欄位**不存在於 `TcContact` 資料表**：

```text
FBotService_TeamsId, FChatWorkGroupId, FIgId, FIgId_APP, FIgId_GW,
FInstagramBind, FMicrosoftId, FTelegramBind, FTelegramId
```

### 修復方式（已執行）

```sql
-- 1. 在 TcContact 新增缺少的欄位
ALTER TABLE TcContact
  ADD COLUMN IF NOT EXISTS FBotService_TeamsId varchar(100) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FChatWorkGroupId varchar(36) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FIgId varchar(100) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FIgId_APP varchar(100) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FIgId_GW varchar(100) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FInstagramBind bit(1) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FMicrosoftId varchar(100) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FTelegramBind bit(1) DEFAULT NULL,
  ADD COLUMN IF NOT EXISTS FTelegramId varchar(100) DEFAULT NULL;

-- 2. 確保 FSourceType = 'local'，FSource = NULL
UPDATE TsField
SET FSourceType = 'local', FSource = NULL
WHERE FUnitId = '00000000-0000-0000-0001-020000001002'
AND FName IN ('FBotService_TeamsId','FChatWorkGroupId','FIgId','FIgId_APP',
              'FIgId_GW','FInstagramBind','FMicrosoftId','FTelegramBind','FTelegramId');
```

如果 `getItem` 又壞了，以上 SQL 重跑一遍即可修復。

---

## 🖥️ UI 元素名稱對照

### 聯絡人列表頁（Ecp.Contact.List.page）

| 元素 | 名稱 |
|------|------|
| 查詢方案 | 請選擇查詢方案 |
| 姓名查詢 | 姓名 textbox |
| 搜尋新增 | 搜尋新增 textbox |
| 查看按鈕 | 查看 |
| 設定密碼 | 設定密碼 |
| 刪除 | 刪除 |
| 重新整理 | 重新整理 |
| 檢查重複 | 檢查重複連絡人 |
| 直接合併 | 直接合併 |
| 右側窗格 | 顯示/隱藏右側閱覽窗格 |

### 聯絡人表單左側頁籤

| 頁籤名稱 | 說明 |
|----------|------|
| 表單 | 基本資訊/私人資訊/管理/登入資訊 |
| 身分 | 帳號身份 |
| 組織架構 | 所屬部門層級 |
| 人際網路 | 關聯人員 |
| 活動 | 相關活動記錄 |
| 線索 | 商業線索 |
| 商機 | 商機記錄 |
| 日誌 | 操作日誌 |
| 授權 | 存取授權 |
| 附件 | 附件檔案 |
| 備註 | 多筆備註記錄（TsNote） |
| 通話記錄 | 電話通話歷史 |
| 聯絡人匯總 | 聯絡人彙整 |
| 服務請求 | 關聯服務請求 |
| 聯絡人重要記事匯總 | 唯讀匯總檢視，無法直接在此新增/編輯 |

### 主系統頂部導航

| 元素 | 名稱 |
|------|------|
| 主選單一級 | 工作檯 / 日常辦公 / 客戶關係 / 系統管理 / 聊天監控 / 智能服務 / 測試功能 |
| 右上角 | 系統消息 / 工時 / 幫助 / 系統管理員（當前用戶） |

---

## 完整建立用戶流程（員工 + 帳號 + 綁定 + 密碼）✅ 實際驗證

ECP 用戶 = 員工(tsuser) + 帳號(tsaccount) + 橋接(tsaccountidentity)，需分開建立再綁定。

> ⚠️ 建立帳號時直接帶 `FPassword` 會存成**明文無法登入**，必須事後用 `modifyPassword` 設定。

### Step 1 — 登入
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.login.data" \
  -c cookies.txt -H "Content-Type: application/json" \
  -d '{"loginName":"James.Huang","password":"<ECP_PASSWORD>","language":"zh-tw"}'
```

### Step 2 — 建立員工
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.User.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data":[{"FName":"趙雲","FGender":"1","FDepartmentId":"00000000-0000-0000-1001-000000000001","FDuty":"蜀漢五虎上將","FEmail":"zhaoyu@threekingdoms.com","FLanguage":"zh-tw","FEnabled":true}]}'
# → {"entityIds":["<員工ID>"]}
```

### Step 3 — 建立帳號
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.save.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"data":[{"FName":"趙雲","FLoginName":"zhaoyu","FLanguage":"zh-tw","FEnabled":true}]}'
# → {"entityIds":["<帳號ID>"]}
```

### Step 4 — 綁定帳號 ↔ 員工
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.AccountIdentity.bind.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"unitId":"00000000-0000-0000-0001-000000001002","accountId":"<帳號ID>","identityTypeId":"564cf69e-76d6-4baf-b584-6e04c2911dae","entityId":"<員工ID>"}'
# → {}
```

### Step 5 — 設定密碼
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.Account.modifyPassword.data" \
  -b cookies.txt -H "Content-Type: application/json" \
  -d '{"accountId":"<帳號ID>","newPassword":"<EXAMPLE_PASSWORD>","requireOldPassword":false}'
# → {}
```

### Step 6 — 驗證登入
```bash
curl -X POST "https://econtact.ai3.cloud/ecp/Qs.OnlineUser.login.data" \
  -c test_cookies.txt -H "Content-Type: application/json" \
  -d '{"loginName":"zhaoyu","password":"<EXAMPLE_PASSWORD>","language":"zh-tw"}'
# → {} = 登入成功
```

### 固定參數速查

| 參數 | 固定值 |
|------|--------|
| `unitId` (綁定用) | `00000000-0000-0000-0001-000000001002` |
| `identityTypeId` (員工) | `564cf69e-76d6-4baf-b584-6e04c2911dae` |
| 集團部門 `FDepartmentId` | `00000000-0000-0000-1001-000000000001` |

---

## Conformance Addendum

## When to Use
ECP (eContact Platform) system knowledge base. Use when working with ECP APIs, UI automation, architecture, workflows, chat, external channel integrations, or employee account management on the Quicksilver Framework.

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
