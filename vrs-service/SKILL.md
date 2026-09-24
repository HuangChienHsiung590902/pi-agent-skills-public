---
name: vrs-service
description: 當使用者提到 VRS、手語視訊、轉譯中心、服務記錄、通話統計、手譯員、登入系統、抓資料、查報表時使用此 skill。協助判斷應執行哪個 VRS command，並提供系統背景知識。
version: 2.0.0
---

# VRS 手語視訊轉譯中心

## 這個 Skill 的作用

當使用者有 VRS 相關需求時，協助判斷該使用哪個 command，並提供系統背景知識讓 Claude 做出正確決策。

> CRM 通話記錄細節另見 `vrs-calllog`。TA101 座席狀態明細表清理／跟服務紀錄比對另見 `vrs-ta101-report`。若需手動登入、選單導覽、或此處腳本流程失效時的備援操作，改用 `vrs` skill（Playwright UI 操作）。

## 何時觸發哪個 Command

| 使用者說 | 應執行 |
|---------|--------|
| 「登入」、「開啟 CRM」、「幫我進系統」 | `/vrs-login` |
| 「抓資料」、「查服務記錄」、「服務統計」 | `/vrs-data` |
| 「工作台通話記錄」、「CRM 通話記錄」、「客戶關係通話」、「今日通話」、「抓歷史通話」 | `/vrs-crm-calllog` |
| 「報表」、「Excel」、「手譯員績效」、「排行」、「更新報表」 | `/vrs-report` |
| 「API」、「呼叫端點」、「token」、「headers 怎麼帶」 | `/vrs-api` |

## 系統背景知識

- **系統名稱**：手語視訊轉譯中心後台系統
- **登入帳號**：CS0006（黃建雄）
- **瀏覽器**：需連線至 Chrome CDP port 9222
- **Token**：登入後存於 `C:\Users\HCH\token.txt`，有效期 3 小時

## 手譯人員帳號規則

帳號格式為 `CS00XX`，例如 CS0006、CS0012、CS0021 等。

## 重要路徑

| 用途 | 路徑 |
|------|------|
| 執行腳本 | `C:\Users\HCH\.claude\skills\vrs-service\scripts\` |
| 報表輸出 | `C:\Users\HCH\.claude\skills\vrs-service\output\` |
| JWT Token | `C:\Users\HCH\token.txt` |
| 暫存目錄 | `C:\Users\HCH\tmp\` |

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
