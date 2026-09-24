---
name: ecp-line-agent
description: 進入 ECP 文字客服台（LINE 客服）替使用者當客服，監控對話、自動回覆用戶訊息。用在：真人客服值班、自動回覆、對話監控。
triggers:
  - 客服
  - line 客服
  - 文字客服
  - 當客服
  - 幫我回覆
argument-hint: "[輪詢間隔秒數，預設60]"
---

# ECP LINE 客服 Skill

## 前置條件

1. Chrome 已用 debug port 9222 啟動（桌面 `啟動Chrome-Debug.ps1`）
2. Playwright MCP 工具可用
3. 已登入 ECP（administrator 帳號，密碼見 `ecp-pwd` skill）

## 完整工作流程

### Step 1：進入文字客服頁面

```javascript
// 點左側選單「文字客服」
() => {
  const items = document.querySelectorAll('#MenuZone .JuiTreeTextCell.JuiTreeLeafIcon');
  for (const el of items) {
    if (el.innerText?.trim() === '文字客服') { el.click(); return 'clicked'; }
  }
  return 'not found';
}
```

等 iframe 載入後，客服台在 `src` 含 `line-proxy` 的 iframe 內。

### Step 2：讀取所有對話房間

```javascript
() => {
  const f = Array.from(document.querySelectorAll('iframe')).find(f => f.src?.includes('line-proxy'));
  if (!f) return [];
  const doc = f.contentDocument;
  return Array.from(doc.querySelectorAll('.room')).map(r => ({
    name:   r.querySelector('.rname')?.innerText?.trim(),
    last:   r.querySelector('.rlast')?.innerText?.trim(),
    closed: r.classList.contains('closed'),
    badge:  r.querySelector('.badge')?.innerText?.trim()
  }));
}
```

- `.closed` = false → 對話開啟中（需要處理）
- `badge` 有值 → 有未讀訊息

### Step 3：切換到開啟中的對話

```javascript
async () => {
  const f = Array.from(document.querySelectorAll('iframe')).find(f => f.src?.includes('line-proxy'));
  const doc = f.contentDocument;
  const openRoom = Array.from(doc.querySelectorAll('.room')).find(r => !r.classList.contains('closed'));
  if (!openRoom) return 'no open room';
  openRoom.click();
  await new Promise(r => setTimeout(r, 800)); // 等訊息載入
  return 'switched';
}
```

### Step 4：讀取對話訊息

```javascript
() => {
  const f = Array.from(document.querySelectorAll('iframe')).find(f => f.src?.includes('line-proxy'));
  const doc = f.contentDocument;
  return Array.from(doc.querySelectorAll('.msg')).map(m => ({
    dir:  m.classList.contains('in') ? 'user' : 'agent',
    text: m.innerText?.trim()
  }));
}
// dir='user' → 用戶來訊，dir='agent' → 客服回覆
```

### Step 5：回覆訊息

```javascript
() => {
  const f = Array.from(document.querySelectorAll('iframe')).find(f => f.src?.includes('line-proxy'));
  const doc = f.contentDocument;
  const ta = doc.querySelector('textarea');
  if (!ta) return 'no textarea';

  // 填入回覆內容
  const nativeSetter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
  nativeSetter.call(ta, '【回覆內容】');
  ta.dispatchEvent(new Event('input', { bubbles: true }));

  // 點送出
  const sendBtn = Array.from(doc.querySelectorAll('button')).find(b => b.innerText?.trim() === '送出');
  if (sendBtn) { sendBtn.click(); return 'sent'; }
  return 'no send btn';
}
```

> 送出後訊息存入 cbm-lite DB，同時 push 到 LINE。

### Step 6：設定輪詢（持續監控）

```
// 每 60 秒用 ScheduleWakeup 重新觸發 skill
// prompt: "你幫我進入工作臺的文字客服中來幫我當一下客服,用戶問啥你來回答他"
```

---

## 完整監控循環邏輯

```
1. 讀所有房間 → 找未 closed 且有 badge 的
2. 切換到該房間
3. 讀最後幾則訊息
4. 找最後一則 dir='user' 的訊息
5. 若此訊息之後沒有 dir='agent' 的訊息 → 需要回覆
6. 根據訊息內容生成回覆（自由發揮，或查知識庫）
7. 送出回覆
8. 截圖確認
9. ScheduleWakeup(60s) 繼續監控
```

---

## DOM 結構速查

| 選擇器 | 說明 |
|--------|------|
| `iframe[src*="line-proxy"]` | 整個客服台 iframe |
| `.room` | 對話房間列表項目 |
| `.room.closed` | 已關閉對話 |
| `.rname` | 房間名稱（用戶名） |
| `.rlast` | 最後一則訊息預覽 |
| `.badge` | 未讀訊息數 |
| `.msg` | 訊息泡泡 |
| `.msg.in` | 用戶發送（inbound） |
| `.msg:not(.in)` | 客服回覆（outbound） |
| `textarea` | 回覆輸入框 |
| `button[送出]` | 送出按鈕 |
| `button[結束服務]` | 結束此次對話 |

---

## 回覆 API（直接 fetch，不透過 UI）

若 DOM 操作不穩定，可直接呼叫後端 API：

```javascript
async () => {
  const r = await fetch('https://hch.james-huang.org/line-proxy/agent/send', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      roomId:  '【roomId】',
      content: '【回覆內容】',
      agent:   '客服'
    })
  });
  return { status: r.status, ok: r.ok };
}
```

取得 roomId：

```javascript
async () => {
  const r = await fetch('https://hch.james-huang.org/line-proxy/agent/rooms');
  const data = await r.json();
  return data.rooms.filter(rm => !rm.closed).map(rm => ({ name: rm.name, roomId: rm.roomId, last: rm.lastMsg }));
}
```

---

## 知識庫回覆（未來擴充）

cbm-lite 提供知識庫搜尋 API：

```javascript
async () => {
  const r = await fetch('https://hch.james-huang.org/line-proxy/cbm/search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query: '【用戶問題】', limit: 3 })
  });
  const data = await r.json();
  return data.records; // 回傳最相關的知識庫段落
}
```

根據搜尋結果組合回覆，再用 send API 送出。

---

## 已知問題與坑

### 1. LINE Push 429 — 本月配額用盡

```
cbm-lite LINE push failed: HTTP 429: {"message":"You have reached your monthly limit."}
```

- **現象**：訊息存入 DB（客服台看得到），但用戶手機 LINE 收不到
- **確認方式**：`grep "push failed\|push ok" C:\com\cbm-lite\logs\cbm-lite.log | tail -5`
- **解法**：到 LINE Official Account Manager 購買追加訊息包，或等下月重置

### 2. Webhook 530 — 暫時性

- **現象**：LINE 送 webhook 給 `gateway.james-huang.org` 時回 530
- **確認方式**：在 LINE Developers Console → HCH channel → Messaging API → 點 **Verify**
- **根本原因**：通常是 cloudflared tunnel 剛重啟瞬間、或 cbm-lite 暫時無回應
- **解法**：確認 `cloudflared.exe` 和 cbm-lite（port 12621）都在跑，再 Verify

### 3. 兩個 LINE Channel

| Channel | ID | 用途 |
|---------|-----|------|
| AI3.5 | 2006519220 | 另一個 bot |
| HCH | 2006734807 | 主要使用的客服 bot，webhook = `gateway.james-huang.org/gateway` |

操作客服時對應的是 **HCH channel**（2006734807）。

### 4. 對話切換後訊息需要等 ~800ms 載入

`openRoom.click()` 後要 `setTimeout(800)` 再讀 `.msg`，否則拿到空陣列。

### 5. textarea 值需用原生 setter 設定

直接 `ta.value = 'xxx'` 不會觸發 React/框架的 input event，要用：
```javascript
const nativeSetter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
nativeSetter.call(ta, text);
ta.dispatchEvent(new Event('input', { bubbles: true }));
```

---

## Conformance Addendum

## When to Use
進入 ECP 文字客服台（LINE 客服）替使用者當客服，監控對話、自動回覆用戶訊息。用在：真人客服值班、自動回覆、對話監控。

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
