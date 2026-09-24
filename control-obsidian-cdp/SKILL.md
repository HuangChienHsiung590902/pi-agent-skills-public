---
name: control-obsidian-cdp
description: 用 Chrome DevTools Protocol (CDP) 直接操作正在執行的 Obsidian 桌面版（點按鈕、填表單、截圖驗證畫面），而不只是用官方 CLI 讀寫檔案。當使用者要求「操作 Obsidian」「測試這個 Obsidian 外掛」「幫我在 Obsidian 裡試試看」「截圖確認 Obsidian 畫面」，或需要驗證自訂 Obsidian 外掛（.obsidian/plugins/ 底下的外掛）的實際 UI 互動是否正常時使用。這是使用者明確指定「以後要控制 Obsidian 就用這個方式」的固定做法，不要改用其他方式。
---

# 用 CDP 直接操控 Obsidian

Obsidian 桌面版是 Electron app，本質上就是一個 Chromium 視窗，天生支援 Chrome DevTools Protocol
（CDP）遠端偵錯——用跟操控真實 Chrome 一樣的協定（Runtime.evaluate 跑 JS、Page.captureScreenshot
截圖）就能直接點按鈕、填表單、讀畫面狀態，比純靠讀程式碼猜測 UI 行為可靠得多，也比傳統座標點擊穩定（CDP 是操作 DOM，不是操作螢幕座標）。

**這是使用者明確要求的固定做法**（2026-07-29：「用 CDP 自動化直接操作了真實的 Obsidian…以後要控制
obsidian就用這方式」）——之後任何需要「實際操作/測試 Obsidian 畫面」的任務，優先用這個 skill，不要
自己重新發明別的方式。

## 跟其他相關 skill 的差異（不要用錯）

- **`connect-chrome` skill**：只接管「真的 Chrome 瀏覽器」在固定 port 9222（`playwright-vrs`，
  寫死在 `~/.claude.json` 的 MCP server），**不能拿來操作 Obsidian**——Obsidian 是獨立的 Electron
  程序，不是那個 port 上的 Chrome。要改接這裡的 CDP port 得改 `~/.claude.json` 設定並重啟 Claude
  Code，太重，不要走這條路。
- **`obsidian-cli` skill**：官方 CLI，只能做文件/筆記層級的操作（建立、讀取、搜尋、tag、
  properties），**沒辦法點擊畫面上的按鈕、開啟 Modal、截圖驗證外掛 UI**。要做這些事才需要本
  skill。
- 兩者可以互補：用 `obsidian-cli` 做批次筆記操作，用本 skill 驗證「畫面上看到的東西對不對」。

## 使用流程

### 1. 用 debug port 重啟 Obsidian

```powershell
powershell -File "C:\Users\HCH\.claude\skills\control-obsidian-cdp\scripts/restart_obsidian_debug.ps1"
# 自訂 port：-Port 9334
```

這會關掉現有 Obsidian（會用 `Stop-Process -Name Obsidian -Force`，使用者未儲存的變更 Obsidian
自己有 auto-save 機制，一般不用擔心，但如果使用者正在編輯敏感內容建議先提醒一聲），用
`--remote-debugging-port=9333 --remote-allow-origins=*` 重新啟動。

⚠ **`--remote-allow-origins=*` 這個參數不能省**——沒有它，CDP WebSocket 連線會被直接拒絕：
```
403 Rejected an incoming WebSocket connection from the http://127.0.0.1:9333 origin.
Use the command line flag --remote-allow-origins=http://127.0.0.1:9333 to allow connections
from this origin or --remote-allow-origins=* to allow all origins.
```

Obsidian 會記住上次開啟的分頁並在重啟後自動還原，這對連續操作/驗證很方便（例如要測試的外掛畫面
本來就開著，重啟後還是開著）。

### 2. 確認 Python 環境（只需裝一次）

```bash
python -c "import websocket" 2>&1 || pip install websocket-client --quiet
```

⚠ 注意套件名稱是 `websocket-client`（import 時用 `import websocket`），**不是**
`websockets`——這台機器預設沒裝 `websockets`，裝錯套件會 import 失敗。`requests` 這台機器已經有裝。

### 3. 用 `scripts/cdp_client.py` 操作/截圖

```bash
# 跑一段 JS，回傳值會印出來（會自動找到目前開著的主視窗，不用手動查 webSocketDebuggerUrl）
python "C:\Users\HCH\.claude\skills\control-obsidian-cdp\scripts/cdp_client.py" eval "document.title"

# 截圖存檔，再用 Read 工具打開看
python "C:\Users\HCH\.claude\skills\control-obsidian-cdp\scripts/cdp_client.py" screenshot "C:\path\to\out.png"
```

### 4. 操作完畢，恢復正常模式

```powershell
powershell -File "C:\Users\HCH\.claude\skills\control-obsidian-cdp\scripts/restart_obsidian_normal.ps1"
```

還給使用者一個沒有 debug port 開著的正常 Obsidian（debug port 對外開放不是使用者平常想要的狀態，
用完務必關掉）。

## 常見操作範例

**點擊按鈕**（用文字比對找按鈕，Obsidian 的 Setting/Modal 按鈕沒有穩定的 id/class 可選）：
```js
(function() {
  const btns = Array.from(document.querySelectorAll('button'));
  const btn = btns.find(b => b.textContent.includes('新增'));
  if (!btn) return 'not found: ' + btns.map(b=>b.textContent).join('|');
  btn.click();
  return 'clicked';
})()
```

**填寫 Setting 元件的輸入框**（⚠ 直接 `input.value = x` 沒用，Obsidian 的元件是監聽真正的
`input` 事件，必須用原生 property setter 再手動 dispatch）：
```js
(function() {
  const items = Array.from(document.querySelectorAll('.setting-item'));
  const item = items.find(el => el.querySelector('.setting-item-name')?.textContent.trim() === '標題');
  const input = item.querySelector('input, textarea');
  const proto = input.tagName === 'TEXTAREA' ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
  Object.getOwnPropertyDescriptor(proto, 'value').set.call(input, '新的值');
  input.dispatchEvent(new Event('input', { bubbles: true }));
  return 'ok';
})()
```

**直接呼叫外掛內部方法，繞過按鈕，快速判斷是後端 bug 還是 UI 接線 bug**（Obsidian 把 `app` 掛在
全域，外掛實例可以直接拿到）：
```js
(async function() {
  const plugin = app.plugins.plugins['<plugin-id>'];
  try {
    const result = await plugin.someInternalClient.someMethod(args);
    return { ok: true, result };
  } catch (e) {
    return { ok: false, error: e.message, stack: e.stack };
  }
})()
```
這招在除錯時特別有用：如果直接呼叫內部方法成功、但按鈕點擊沒反應，代表問題在 UI 接線（事件綁定/
DOM 選錯元素）；如果直接呼叫也失敗，代表問題在外掛本身或它呼叫的後端。

**開啟設定頁裡特定外掛的分頁**：
```js
app.setting.open();
app.setting.openTabById('<plugin-id>');
```

## 已知踩坑

1. **`403 Rejected...`**：忘記加 `--remote-allow-origins=*`，見上方步驟 1。
2. **`ModuleNotFoundError: No module named 'websockets'`**：裝錯套件名稱，要裝 `websocket-client`
   （import 名稱是 `websocket`），不是 `websockets`。
3. **Windows 主控台印 emoji/中文會噴 `UnicodeEncodeError: 'cp950' codec can't encode...`**：
   `scripts/cdp_client.py` 已經在檔案開頭加了 `sys.stdout.reconfigure(encoding="utf-8")` 處理過，若自己
   另外寫類似腳本記得也要加，不然只要 JS 回傳值裡有 emoji（例如按鈕文字含 🤖）就會直接噴例外。
4. **長時間非同步操作（例如按鈕觸發 LLM 呼叫）逾時**：`websocket.create_connection` 預設的
   `timeout` 太短會在等回應時炸 `WebSocketTimeoutException`。`scripts/cdp_client.py` 已經設成 60 秒；
   如果單次 `Runtime.evaluate` 裡面又寫了輪詢迴圈（例如等按鈕文字變回原狀），迴圈總長度不要超過
   這個 timeout，或是分成多次呼叫用 `sleep` + 重新截圖的方式代替單次長迴圈。
5. **表單填完看起來沒生效**：多半是用了 `el.value = x` 而不是原生 setter + dispatchEvent，見上方
   「填寫 Setting 元件的輸入框」範例。
6. **重啟後找不到視窗/target**：`scripts/cdp_client.py` 已經自動抓 `type=="page"` 的第一個 target，如果
   環境裡真的有多個 Obsidian 視窗（少見）才需要手動用 `http://127.0.0.1:<port>/json` 查完整清單
   自己選。

## 這個技巧不限 Obsidian

任何 Electron app 都適用同一套手法（`--remote-debugging-port` + `--remote-allow-origins=*` +
CDP over WebSocket），不是 Obsidian 專屬的。要操作別的 Electron 桌面 app 時可以直接照搬這套流程。

---

## Conformance Addendum

## When to Use
用 Chrome DevTools Protocol (CDP) 直接操作正在執行的 Obsidian 桌面版（點按鈕、填表單、截圖驗證畫面），而不只是用官方 CLI 讀寫檔案。當使用者要求「操作 Obsidian」「測試這個 Obsidian 外掛」「幫我在 Obsidian 裡試試看」「截圖確認 Obsidian 畫面」，或需要驗證自訂 Obsidian 外掛（.obsidian/plugins/ 底下的外掛）的實際 UI 互動是否正常時使用。這是使用者明確指定「以後要控制 Obsidian 就用這個方式」的固定做法，不要改用其他方式。

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
