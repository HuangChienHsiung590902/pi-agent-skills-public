---
name: vrs-call-export
description: 從「手語視訊轉譯中心值機系統」(sfaa-vrsp-ecp.qbicloud.com) 的通話記錄，抓取該通的「錄音檔(mp3)」與「即時語音轉文字逐字稿」並存檔。當使用者要求「抓錄音」、「下載音檔」、「匯出逐字稿/轉譯文字」、「把這通對話存下來」、「錄音加逐字稿一起抓」時使用。透過已接管的 Chrome (Playwright MCP, port 9222) 操作，不自行登入或開新 Chrome。
---

# VRS 通話錄音 + 逐字稿匯出

把「手語視訊轉譯中心值機系統」某一通通話的**錄音檔**與**逐字稿**抓出來存到本機。

## 名詞與頁面結構（實測）

- 系統首頁：`https://sfaa-vrsp-ecp.qbicloud.com/ecp/Qs.MainFrame.page`
- **通話記錄列表**在 iframe `Ecp.CallLog.List.page`，欄位：聯絡人 / 進線種類 / 主叫號 / 被叫號 / 通話秒數 / … / 電話唯一辨識值。
  - ⚠️「聯絡人」欄是**超連結**，雙擊它只會開「聯絡人資料卡」，**不是**通話明細。
  - 要雙擊**同一列的非連結欄位**（例如「通話秒數」格）才會在新分頁開啟**通話明細**。
- **通話明細頁**左側有三個子頁：`表單` / `即時語音轉文字記錄` / `日誌`。
  - `表單` 子頁右上工具列有 **「錄音播放」** 與 **「下載音檔」**（class `JuiButtonText`）。
  - `即時語音轉文字記錄` 子頁是逐字稿，訊息結構：
    - 講者 `.ChatMessageSenderName`（VR_xxx＝聽障者端訊息來源、CSxxxx＝手譯員）
    - 內容 `.ChatMessageContent`（文字含「語音轉文字訊息(聽人/手譯員): …」）
    - 時間 `.ChatMessageTime`（HH:MM:SS）
- 自動下載的 mp3 會落在 `C:\Users\HCH\.playwright-mcp\`，檔名尾段即該通的**電話唯一辨識值**。

## 鐵則

- **只接管現有 Chrome**（沿用 `connect-chrome` 的規則）：port 9222 沒開就請使用者自己用 debug 捷徑開，不要自動啟動 Chrome。
- 這是 **[PROD] 正式系統**：只做「讀取／下載」，**不要**點任何會修改、刪除、保存的按鈕。

---

## 步驟

### 0. 確認已接管 Chrome

若這個 session 還沒接管，先跑 `/connect-chrome`（或確認 port 9222 已開、Playwright MCP 已連上）。
確認目前在 `Qs.MainFrame.page`，且通話記錄列表（`通話記錄`分頁）開著。

### 1. 開啟目標通話的明細頁

在通話記錄列表雙擊目標列的**非連結欄位**。下面的 helper 會：在 `CallLog.List` 表格中，找到「包含 `matchText` 的列」，並對該列「不含超連結的儲存格」送出雙擊，避免誤觸聯絡人連結。

`matchText` 可用該列任一可辨識文字：被叫號、通話秒數、或電話唯一辨識值。若不指定（傳空字串），預設開「目前被選取(高亮)的列」或第一列。

```js
// browser_evaluate
(() => {
  const matchText = "<<MATCH>>"; // ← 換成被叫號/秒數/電話唯一辨識值；留空字串則取選取列或第一列
  const docs=[];(function c(d){docs.push(d);[...d.querySelectorAll('iframe')].forEach(f=>{try{if(f.contentDocument)c(f.contentDocument);}catch(e){}});})(document);
  const d=docs.find(d=>{try{return /CallLog\.List/.test(d.defaultView.frameElement?.src||'')}catch(e){return false}}) || docs.find(d=>d.querySelector('tr'));
  if(!d) return 'grid not found';
  let rows=[...d.querySelectorAll('tr')].filter(r=>r.querySelector('td'));
  let row;
  if(matchText.trim()){
    row=rows.find(r=>r.textContent.includes(matchText.trim()));
  }else{
    row=rows.find(r=>/selected|active|hover|current/i.test(r.className)) || rows[0];
  }
  if(!row) return 'row not matched: '+matchText;
  // 選一個不含 <a> 連結的儲存格雙擊（避開聯絡人連結欄）
  const cell=[...row.querySelectorAll('td')].find(td=>!td.querySelector('a') && td.offsetParent) || row;
  const r=cell.getBoundingClientRect();
  const win=d.defaultView;
  const opt={bubbles:true,cancelable:true,view:win,clientX:r.x+5,clientY:r.y+5};
  for(const ev of ['mousedown','mouseup','click','mousedown','mouseup','click','dblclick'])
    cell.dispatchEvent(new MouseEvent(ev,opt));
  return 'double-clicked row: '+row.textContent.trim().slice(0,60);
})()
```

截圖確認新分頁已開、出現「通話記錄: I…」明細頁。

### 2. 下載錄音檔（先做，因為要切到「表單」子頁）

先切到 `表單` 子頁，再點「下載音檔」。

```js
// browser_evaluate ── 切到「表單」子頁
(() => {
  const docs=[];(function c(d){docs.push(d);[...d.querySelectorAll('iframe')].forEach(f=>{try{if(f.contentDocument)c(f.contentDocument);}catch(e){}});})(document);
  for(const d of docs){
    const all=[...d.querySelectorAll('*')];
    if(!all.some(e=>e.children.length===0 && e.textContent.trim()==='即時語音轉文字記錄')) continue; // 確認是明細頁的左選單
    const form=all.find(e=>e.children.length===0 && e.textContent.trim()==='表單');
    if(form){ form.click(); return 'switched to 表單'; }
  }
  return 'detail-page nav not found';
})()
```

```js
// browser_evaluate ── 點「下載音檔」（會觸發瀏覽器下載 mp3）
(() => {
  const docs=[];(function c(d){docs.push(d);[...d.querySelectorAll('iframe')].forEach(f=>{try{if(f.contentDocument)c(f.contentDocument);}catch(e){}});})(document);
  for(const d of docs){
    const cand=[...d.querySelectorAll('a,div,span,button')].filter(e=>/下載音檔/.test(e.textContent)&&e.textContent.replace(/\s/g,'').length<=8&&e.offsetParent);
    if(cand.length){ cand.sort((a,b)=>a.textContent.length-b.textContent.length); cand[0].click(); return 'clicked 下載音檔'; }
  }
  return '下載音檔 not found（確認已在「表單」子頁）';
})()
```

mp3 會被下載到 `C:\Users\HCH\.playwright-mcp\`。用 Bash/Glob 找出剛下載、最新的 `*.mp3` 取得實際檔名（檔名尾段＝電話唯一辨識值），作為這通的基準檔名 `BASENAME`：

```bash
ls -t /c/Users/HCH/.playwright-mcp/*.mp3 2>/dev/null | head -1
```

### 3. 抓逐字稿

切到 `即時語音轉文字記錄` 子頁並擷取訊息。

```js
// browser_evaluate ── 切到「即時語音轉文字記錄」
(() => {
  const docs=[];(function c(d){docs.push(d);[...d.querySelectorAll('iframe')].forEach(f=>{try{if(f.contentDocument)c(f.contentDocument);}catch(e){}});})(document);
  for(const d of docs){
    const el=[...d.querySelectorAll('*')].find(e=>e.children.length===0 && e.textContent.trim()==='即時語音轉文字記錄');
    if(el){ el.click(); return 'switched to transcript'; }
  }
  return 'transcript nav not found';
})()
```

等 1~2 秒讓泡泡載入，再擷取（存成檔案再讀，逐字稿可能很長）：

```js
// browser_evaluate（建議帶 filename: "vrs-transcript.json" 存檔再讀）
(() => {
  const docs=[];(function c(d){docs.push(d);[...d.querySelectorAll('iframe')].forEach(f=>{try{if(f.contentDocument)c(f.contentDocument);}catch(e){}});})(document);
  let td=null; for(const d of docs){ if(d.querySelector('.ChatMessageContent')) td=d; }
  if(!td) return {error:'transcript frame not found'};
  const header=(td.body.innerText.split('\n').find(l=>l.trim())||'').trim(); // 例：CS0014,VR_0800020021
  const msgs=[...td.querySelectorAll('.ChatMessageContent')].map(c=>{
    let blk=c; for(let i=0;i<6&&blk.parentElement;i++){ blk=blk.parentElement; if(blk.querySelector('.ChatMessageSenderName')&&blk.querySelector('.ChatMessageTime')) break; }
    const name=((blk.querySelector('.ChatMessageSenderName')||{}).textContent||'').trim();
    const time=((blk.querySelector('.ChatMessageTime')||{}).textContent||'').trim();
    const content=c.textContent.trim();
    return {name,time,content};
  });
  return {header, count:msgs.length, msgs};
})()
```

### 4. 存檔

把逐字稿排成易讀文字檔，與 mp3 放同一個輸出資料夾。

- 輸出資料夾預設：`C:\Users\HCH\VRS匯出\<BASENAME>\`（`BASENAME`＝步驟 2 的 mp3 去副檔名；其尾段即電話唯一辨識值）
- 用 **Write** 工具寫 `逐字稿.txt`：每行 `[HH:MM:SS] 講者: 內容`，檔頭放電話唯一辨識值與通話雙方。
- 用 **Bash** 把步驟 2 的 mp3 從 `.playwright-mcp\` 搬進輸出資料夾。

逐字稿建議格式：

```
電話唯一辨識值: <BASENAME 尾段>
通話雙方: <header，例 CS0014,VR_0800020021>
訊息數: <count>
----
[13:25:27] VR_0800020021: 語音轉文字訊息(聽人): 嗯。
[13:25:30] VR_0800020021: 語音轉文字訊息(聽人): 歡迎並感謝您致電APPLE。
...
```

完成後回報：輸出資料夾路徑、mp3 檔名與大小、逐字稿行數。

---

## 批次匯出（API 快速法 — 大量首選）

逐筆點 UI 一通約 12–20 秒，上千通要數十小時。系統其實有現成 JSON API，可帶登入 cookie 直接抓，速度與穩定度遠勝 UI。本資料夾附了可直接用的 Node 腳本（`scripts/lib.mjs` / `scripts/collect.mjs` / `scripts/export.mjs`，Node 18+，免安裝套件）。

**三個關鍵 API（同源 POST，body 為 JSON 字串、`content-type: text/plain`）：**
- 清單分頁：`qsvd-list/Ecp.CallLog.getListData.data`
  body `{pageIndex, isRefresh:false, listId, keyword:"", showPageCount:true}`
  → `data.records[]`（每筆含 `FId`＝entityId、`FCallerPIN`＝電話唯一辨識值、`FStartTime/FEndTime/FAni/FDnis/FDuration/FUserId$/FContactId$`）、`data.pageCount`、`data.totalSize`。每頁 25 筆、`FStartTime` 降冪。
- 逐字稿：`Ecp.ChatHistory.getChatItems.data`
  body `{masterUnitId, masterEntityId: FId}`、header `qs-pagecode: Ecp.ChatRoom.CallLogChatHistoryOnline`
  → `rooms[].recentMessages[]`（`senderName / sendMillis / content`）。
- 音檔：`Ecp.VRM.downloadAudio.data` body `{entityId: FId}` → `{servletUrl}`；再 GET `ORIGIN+servletUrl` 取 mp3（`application/octet-stream`）。

**常數**（會隨系統改版/不同清單而異，必要時重新從 DevTools 網路面板抓）：
- `ORIGIN = https://sfaa-vrsp-ecp.qbicloud.com/ecp/`
- `LIST_ID`（getListData 的 listId）、`MASTER_UNIT`（getChatItems 的 masterUnitId）— 見 `scripts/lib.mjs`，可從一次真實請求的 body 取得。

**登入 cookie（httpOnly）取得**：`JSESSIONID` 是 httpOnly，`document.cookie` 讀不到。`scripts/lib.mjs` 透過 debug Chrome 的 CDP（`http://127.0.0.1:9222`）呼叫 `Storage.getCookies` 取出（含 httpOnly），組成 Cookie 標頭。→ 必須先用 `/connect-chrome` 開著 debug Chrome 並已登入。

**執行步驟：**
```bash
cd <本 skill 資料夾>
# 1) 收集某年全部記錄 → records.json
YEAR=2026 node scripts/collect.mjs
# 2) 全量匯出（mp3 + 逐字稿.txt，平放一層；可重跑續傳，已存在自動跳過）
OUTDIR='C:/Users/HCH/VRS匯出_2026' node scripts/export.mjs
#   也可 LIMIT=20 先試跑；OUTDIR/RECORDS 可自訂
```
輸出：`OUTDIR\<電話唯一辨識值>.mp3` 與 `.txt`，另有 `_index.json`（索引）、`_progress.log`、`_errors.log`。秒數為 0 者通常無錄音，只出 txt。長時間執行 cookie 失效時，腳本會自動重抓 cookie 續跑。

> ⚠️ 仍是 PROD 系統：腳本只做 GET/讀取與下載，請維持每筆間的小延遲（預設 120ms）避免造成負載。

## 批次匯出（UI 法 — 少量備援）

若不便用 API，少量可逐筆走 UI：在列表把要抓的列辨識文字（被叫號/秒數/電話唯一辨識值）列出，對每通重複「步驟 1→4」，處理完關掉明細分頁再開下一通：

```js
// browser_evaluate ── 關掉目前作用中的通話明細分頁
(() => {
  const close=[...document.querySelectorAll('.JuiTabStripTabClose')].filter(e=>e.offsetParent);
  if(close.length){ close[close.length-1].click(); return 'closed a detail tab'; }
  return 'no closable tab';
})()
```

## 疑難排解

- 雙擊後跳出的是「聯絡人資料卡」而非通話明細 → 雙擊到「聯絡人」連結欄了，改雙擊同列其他欄位（步驟 1 的 helper 已自動避開含 `<a>` 的儲存格）。
- 找不到「下載音檔」 → 多半是還停在「即時語音轉文字記錄」子頁；先跑步驟 2 切到「表單」再點。
- 逐字稿 `count` 為 0 → 泡泡還沒載入完，等 1~2 秒重跑擷取；或這通本來就沒有轉譯內容。
- mp3 沒出現在 `.playwright-mcp\` → 確認 Playwright 下載沒被瀏覽器攔截；可改用「錄音播放」確認該通是否有錄音。

---

## Conformance Addendum

## When to Use
從「手語視訊轉譯中心值機系統」(sfaa-vrsp-ecp.qbicloud.com) 的通話記錄，抓取該通的「錄音檔(mp3)」與「即時語音轉文字逐字稿」並存檔。當使用者要求「抓錄音」、「下載音檔」、「匯出逐字稿/轉譯文字」、「把這通對話存下來」、「錄音加逐字稿一起抓」時使用。透過已接管的 Chrome (Playwright MCP, port 9222) 操作，不自行登入或開新 Chrome。

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
