---
name: ecp-aipower-ui-map
description: Use when orienting inside the local aipower/ECP web UI, finding where CRM, phone support, service consultation, AIFF, API, menu, account, parameter, workflow, or system administration pages live.
---

# ECP Aipower UI Map

## Overview

This is the working UI map for the local aipower/ECP instance explored on `http://127.0.0.1:22821/aipower/Qs.MainFrame.page`. Use it to avoid rediscovering the left menu structure before changing or testing features.

Related skills: `ecp-lab2-instance`, `ecp-browser-test`, `connect-chrome`, `ecp-pwd`, `ecp-menu`.

Detailed structure snapshot: see `structure-snapshot-2026-07-07.md` in this skill directory.

## Current Access Notes

- Local URL: `http://127.0.0.1:22821/aipower/Qs.MainFrame.page`
- Browser control originally used a dedicated CDP setup against the now-defunct `C:\com\chainsea`
  install — that install and its dedicated MCP server no longer exist; use the current live
  instance's own CDP setup instead, e.g. `ecp-lab2-instance` on port 22821, and the shared
  `playwright-vrs` server on **9222** per `connect-chrome` unless a dedicated server is set up for
  that instance.
- ECP page title: `Aipower`.
- Administrator account seen in UI: `administrator` / `111111` after reset.

## How To Inspect Without Mouse Coordinates

Use DOM clicks on ECP menu items instead of screen coordinates:

```js
const items = document.querySelectorAll('#MenuZone .JuiTreeTextCell.JuiTreeLeafIcon')
items[53].click()
```

Read opened internal tabs from iframe bodies:

```js
[...document.querySelectorAll('iframe.JuiTabStripBody')].map(f => ({
  src: f.src,
  text: f.contentDocument?.body?.innerText?.trim().slice(0, 300)
}))
```

## Main Menu Groups

The explored instance has 8 top-level outlook groups and 132 leaf items.

| Group | Purpose |
|---|---|
| 快捷工作檯 | Workbench shortcut and homepage setup |
| 辦公自動化 | OA: documents, knowledge, exams, discussions, tasks, activities, workflow |
| 通用功能 | Calendar, reports, charts, system messages, login log, marquee |
| 電話客服 | Phone support call records |
| 客戶關係管理 | CRM: companies, contacts, marketing, sales, service, products |
| 基礎設定 | Organization, accounts, logs, parameters, timers, templates, workflow basics |
| 進階設定 | Unit/page/report/server/database/API/permission/developer settings |
| 開發環境項目 | AIFF application management and external applications |

## Important Feature Locations

| Need | Menu Path / Page |
|---|---|
| Call records / phone support | `電話客服 -> 通話記錄` / `Ecp.CallLog.List.page` |
| CRM company master data | `客戶關係管理 -> 企業` / `Ecp.Customer.List.page` |
| Contacts | `客戶關係管理 -> 聯絡人 -> 聯絡人主列表` / `Ecp.Contact.List.page` |
| Marketing plans | `客戶關係管理 -> 行銷 -> 行銷計畫` / `Ecp.MarketPlan.List.page` |
| Quotes | `客戶關係管理 -> 銷售 -> 報價單` / `Ecp.Quote.List.page` |
| Service consultation | `客戶關係管理 -> 服務 -> 服務諮詢` / `Aipower.ServiceConsult.List.page` |
| Accounts/password admin | `基礎設定 -> 帳號管理 -> 帳號` / `Qs.Account.List.page` |
| System parameters | `基礎設定 -> 參數管理 -> 系統參數` / `Qs.Parameter.System.page` |
| Workflow management | `基礎設定 -> 工作流管理 -> 流程管理` / `Wf.Process.List.page` |
| Unit metadata | `進階設定 -> 單元 -> 單元設定` / `Qs.Unit.List.page` |
| Execute SQL | `進階設定 -> 資料庫 -> 執行 SQL` / `Qs.SystemTool.SqlExecute.page` |
| API tokens | `進階設定 -> 資料整合 -> Token 設定` / `Qs.TokenConfig.List.page` |
| Menu editor | `進階設定 -> 其它 -> 功能表設定` / `Qs.Menu.List.page` |
| AIFF apps | `開發環境項目 -> AIFF應用管理` / `Aipower.Aiff.List.page` |
| External apps | `開發環境項目 -> 外部應用` / `Aipower.ExternalApplication.List.page` |

## Representative Page Findings

| Item | Observed Content |
|---|---|
| 文檔 | Document list with download, open, upload, delete |
| 工作項 | Workflow work-item list; empty in this fresh instance |
| 通話記錄 | Call log columns include caller/callee, duration, service start/end, agent, seat |
| 企業 | Company list with owner, department, phone, contact, industry, address |
| 聯絡人主列表 | Contact list with service account, tag, mobile, type, enabled, joined time |
| 服務諮詢 | Service consultation list with status, subject, customer, service request, opportunity, agent/team |
| 帳號 | Shows `系統管理員 / administrator`; has set-password and enable/disable actions |
| 系統參數 | Large settings page: login, attachments, Socket.IO, password policy, CTI, text service, LIFF, service request |
| 單元設定 | Underlying unit metadata, including AIFF and point-activity units |
| 執行 SQL | Page says `該功能已被禁用。` |
| Token 設定 | Has `User` token with 1800-second timeout |
| AIFF應用管理 | Existing AIFF app rows include login/chat related embedded apps |
| 外部應用 | External application binding list; appears empty in this instance |

## Practical Guidance

- For LINE/text-service/AI work, start from `服務諮詢`, `通話記錄`, `聯絡人主列表`, `系統參數`, `AIFF應用管理`, and `Token 設定`.
- For adding or moving menu entries, use `功能表設定`; for DB-level shortcut edits use `ecp-menu`.
- For adding backend units/pages/actions, inspect `單元設定`, `頁面設定`, `本地 API`, `遠端 API`, and related Java module skills.
- Avoid opening all 132 menu items at once; ECP opens internal tabs and can become cluttered. Prefer sampling or click-read-close loops.

## Known Caveats

- `執行 SQL` is disabled in the UI.
- Some menu leaves are hidden until their group is expanded; use DOM enumeration rather than visible text only.
- Internal content is mostly loaded in same-origin iframes under `iframe.JuiTabStripBody`.
- The current map is a snapshot and should be updated as new modules/menus are added.

---

## Conformance Addendum

## When to Use
Use when orienting inside the local aipower/ECP web UI, finding where CRM, phone support, service consultation, AIFF, API, menu, account, parameter, workflow, or system administration pages live.

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
