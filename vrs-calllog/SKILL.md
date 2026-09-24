---
name: vrs-calllog
description: 當使用者提到「CRM 通話記錄」、「工作台通話」、「客戶關係通話記錄」、「今日通話」、「ECP 通話」、「CRM 報表」、「翻頁抓歷史」時使用此 skill。
version: 2.0.0
---

# VRS CRM 通話記錄

## 這個 Skill 的作用

查詢 ECP 系統「客戶關係 → 通話記錄」的通話資料，支援完整歷史抓取（pageIndex 翻頁）與 Excel 報表產出。

> 其他 VRS 需求（登入、服務記錄、統計報表、API）見 `vrs-service` 總覽。手動 UI 操作備援見 `vrs` skill。

## 何時觸發哪個 Command

| 使用者說 | 應執行 |
|---------|--------|
| 「今日通話」、「CRM 通話記錄」、「看一下今天通話」 | `/vrs-crm-calllog` |
| 「抓歷史」、「抓全部」、「2025 通話記錄」、「更新歷史」 | `/vrs-crm-calllog`（選 C 完整歷史）|
| 「CRM 報表」、「通話記錄 Excel」、「產報表」 | `/vrs-crm-calllog`（選 D Excel 選項）|

## 系統背景知識

- **資料來源**：ECP 系統（`sfaa-vrsp-ecp.qbicloud.com`），與 VRS API 分開
- **認證方式**：瀏覽器 session cookie，不是 JWT Token
- **API 端點**：POST `qsvd-list/Ecp.CallLog.getListData.data`，body 為 JSON
- **翻頁機制**：POST body 帶 `pageIndex`（從 1 開始），每頁固定 25 筆
- **listId**：`761c6e12-2247-4cdd-a0f9-93a4a4cb21bc`（固定值）
- **去重機制**：以 `FSessionId` 為唯一鍵

## 重要路徑

| 用途 | 路徑 |
|------|------|
| 完整歷史抓取 | `scripts\scripts/fetch_all_pages.js` |
| 當日快速查看 | `scripts\scripts/fetch_crm_calllog.js` |
| Excel 產生 | `scripts\scripts/gen_excel_crm_history.py` |
| 歷史資料 JSON | `output\crm_calllog_history.json` |
| 2025 Excel | `output\CRM通話記錄統計_2025全年.xlsx` |
| 2026 Excel | `output\CRM通話記錄統計_2026累計.xlsx` |

---

## Conformance Addendum

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
