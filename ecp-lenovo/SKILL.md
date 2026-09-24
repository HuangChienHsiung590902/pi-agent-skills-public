---
name: ecp-lenovo
version: 1.0.0
description: ECP (eContact Platform) system for Lenovo environment. Use when working with ECP on 10.145.119.64:12821, including UI automation, employee management, workflow, chat, and AI services on the Quicksilver Framework.
description_zh: ECP（eContact Platform）Lenovo 環境系統。用於 10.145.119.64:12821 上的 ECP API 呼叫、UI 自動化、員工帳號管理、工作流引擎、文字客服與 AI 服務。
---

## Skill: ecp-lenovo

ECP (eContact Platform) Lenovo 環境專用技能。

---

## 環境資訊

| 項目 | 值 |
|------|-----|
| **URL** | `http://10.145.119.64:12821/ecp/` |
| **帳號** | `Administrator` |
| **密碼** | `<ECP_PASSWORD>` |
| **容器** | `ecp-unlock` (port 12821/12822), `ecp-mariadb` (port 3306) |
| **資料庫密碼** | `<DB_PASSWORD>` |

---

## 快速開始：登入 ECP

```bash
# Step 1: 開瀏覽器
playwright-cli open --browser chrome http://10.145.119.64:12821/ecp/Qs.OnlineUser.Login.page

# Step 2: 填帳號（用 eval 避免 DOM 問題）
playwright-cli eval "() => { document.querySelectorAll('input')[0].value = 'administrator'; }"

# Step 3: 填密碼
playwright-cli eval "() => { document.querySelectorAll('input')[1].value = '<ECP_PASSWORD>'; }"

# Step 4: 點登入
playwright-cli click "登 入"
```

---

## 常用頁面 URL

### 系統管理
| 功能 | URL |
|------|-----|
| 員工 | `http://10.145.119.64:12821/ecp/Qs.User.List.page` |
| 部門 | `http://10.145.119.64:12821/ecp/Qs.Department.List.page` |
| 帳號 | `http://10.145.119.64:12821/ecp/Qs.Account.List.page` |
| 角色 | `http://10.145.119.64:12821/ecp/Qs.Role.List.page` |
| 線上使用者 | `http://10.145.119.64:12821/ecp/Qs.OnlineUser.List.page` |
| 登入日誌 | `http://10.145.119.64:12821/ecp/Qs.LoginLog.List.page` |
| SQL 執行 | `http://10.145.119.64:12821/ecp/Qs.SystemTool.SqlExecute.page` |

### CRM 客戶關係
| 功能 | URL |
|------|-----|
| 企業客戶 | `http://10.145.119.64:12821/ecp/Ecp.Customer.List.page` |
| 聯絡人 | `http://10.145.119.64:12821/ecp/Ecp.Contact.List.page` |
| 線索 | `http://10.145.119.64:12821/ecp/Ecp.Lead.List.page` |
| 商機 | `http://10.145.119.64:12821/ecp/Ecp.Opportunity.List.page` |
| 訂單 | `http://10.145.119.64:12821/ecp/Ecp.Order.List.page` |
| 合約 | `http://10.145.119.64:12821/ecp/Ecp.Contract.List.page` |

### 服務請求與任務
| 功能 | URL |
|------|-----|
| 服務請求 | `http://10.145.119.64:12821/ecp/Ecp.ServiceRequest.List.page` |
| 任務 | `http://10.145.119.64:12821/ecp/Ecp.Task.List.page` |
| 專案 | `http://10.145.119.64:12821/ecp/Ecp.Project.List.page` |

### 聊天客服
| 功能 | URL |
|------|-----|
| 文字客服群組 | `http://10.145.119.64:12821/ecp/Ecp.ChatWorkGroup.List.page` |
| 罐頭訊息 | `http://10.145.119.64:12821/ecp/Ecp.ChatBoxMessage.List.page` |
| 聊天紀錄 | `http://10.145.119.64:12821/ecp/Ecp.ChatLog.List.page` |
| 敏感詞設定 | `http://10.145.119.64:12821/ecp/Ecp.Chat.ChatSensitiveWord.List.page` |

### AI 服務
| 功能 | URL |
|------|-----|
| Copilot 主頁 | `http://10.145.119.64:12821/ecp/KMCopilot/main/main.html` |
| AI 問答任務 | `http://10.145.119.64:12821/ecp/Ecp.AskGptTask.List.page` |
| Prompt 管理 | `http://10.145.119.64:12821/ecp/Ecp.Prompt.List.page` |
| AI Agent | `http://10.145.119.64:12821/ecp/Ecp.AiAgentEntity.List.page` |

### 工作流
| 功能 | URL |
|------|-----|
| 工作項 | `http://10.145.119.64:12821/ecp/Wf.WorkItem.List.page` |
| 流程定義 | `http://10.145.119.64:12821/ecp/Wf.Workflow.List.page` |
| 流程實例 | `http://10.145.119.64:12821/ecp/Wf.Process.List.page` |

---

## UI 自動化技巧

選中清單行／3 步驟刪除記錄／移除遮罩這幾個通用 Playwright UI 操作技巧（`JuiListLeftTable`、
`JuiMessageBox`、`Determine` 按鈕等 class 名稱在各 ECP 環境都通用）**見 `ecp` skill 的
「ECP 清單頁面架構與 UI 刪除流程」章節**，那邊有更完整的脈絡（`JuiMessageBox` vs `JuiDialog`
差異對照表、刪除後驗證步驟），這裡不重複貼一份容易跟著漂移的副本。本環境唯一的差異只是
URL 前綴換成 `10.145.119.64:12821`。

---

## API 呼叫範例

### 新增員工
```javascript
// 在已登入的 ECP 分頁執行：
browser_run_code(code="async (page) => {
  const post = (path, body) => page.evaluate(({path, body}) =>
    fetch('http://10.145.119.64:12821/ecp/' + path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body)
    }).then(r => r.json()), {path, body}
  );

  // 建立員工
  const userRes = await post('Qs.User.save.data', {
    data: [{
      FName: '王小明',
      FGender: '1',
      FDepartmentId: '00000000-0000-0000-1001-000000000001',
      FDuty: '工程師',
      FEmail: 'wang@example.com',
      FLanguage: 'zh-tw',
      FEnabled: true
    }]
  });
  return userRes;
}")
```

### 修改密碼
```javascript
browser_run_code(code="async (page) => {
  const post = (path, body) => page.evaluate(({path, body}) =>
    fetch('http://10.145.119.64:12821/ecp/' + path, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify(body)
    }).then(r => r.json()), {path, body}
  );

  await post('Qs.Account.modifyPassword.data', {
    accountId: '帳號ID',
    newPassword: '<EXAMPLE_PASSWORD>',
    requireOldPassword: false
  });
}")
```

---

## Docker 操作

```bash
# 查看容器狀態
ssh hch@10.145.119.64 "docker compose -f ~/ecp_docker/docker-compose.yml ps"

# 重啟服務
ssh hch@10.145.119.64 "cd ~/ecp_docker && docker compose restart app"

# 匯入 SQL
ssh hch@10.145.119.64 "docker exec -i ecp-mariadb mysql -uroot -p<DB_PASSWORD> ecp < ~/ecp_docker/ecp.sql"

# 完整重建
ssh hch@10.145.119.64 "cd ~/ecp_docker && docker compose down -v && docker compose up -d"
```

---

## 錯誤排除

| 問題 | 解法 |
|------|------|
| `ERR_CERT_AUTHORITY_INVALID` | 使用 `http://10.145.119.64:12821` 而非 `https` |
| 點擊逾時 | 先移除 mask，再用 eval 點擊 |
| session 過期 | 重新整理頁面，重新填帳號密碼 |
| 找不到元素 | 使用 `playwright-cli snapshot` 取得最新 ref |

---

## Conformance Addendum

## When to Use
ECP (eContact Platform) system for Lenovo environment. Use when working with ECP on 10.145.119.64:12821, including UI automation, employee management, workflow, chat, and AI services on the Quicksilver Framework.

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
