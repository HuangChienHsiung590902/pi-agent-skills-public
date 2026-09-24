---
name: cbm-agent-ai-suggest
description: LINE 文字客服台 AI 建議回覆功能 — /agent/suggest、知識庫整合、Markdown 預覽、AiProxy tool-calling 架構，改動橫跨 CbmLiteServer.java 與 AiProxyLlmRunnerImpl.java
triggers:
  - AI建議
  - ai suggest
  - 自動建議
  - 客服模式切換
  - 重新生成
  - 客服AI
  - agent suggest
  - 建議回覆
  - 知識庫整合
  - search_knowledge_base
argument-hint: "[mode|suggest|ui|kb|compile]"
---

# cbm-agent-ai-suggest Skill

LINE 文字客服台（`/agent`）的 **AI 建議回覆**功能：
客戶傳訊 → AI 自動生成建議並填入輸入框（Markdown 渲染） → 客服審閱後手動送出。

姊妹 skill：**cbm-lite**（架構、proxy、Cloudflare、編譯/重啟，已合併原本的 cbm-agent-console/cbm-line/cbm-line-agent）

---

## 架構概覽（重要：兩個不同的 Java 程序）

```
瀏覽器（/agent 頁面）
  │  每 2 秒 loadMessages()
  │    └─ 偵測到新 Outer 訊息 → checkAndSuggest()
  │         └─ GET /agent/suggest?roomId=xxx
  │              └─ CbmLiteServer (port 12621)
  │                   └─ POST http://127.0.0.1:22821/aipower/v1/chat/completions
  │                        └─ Tomcat aipower webapp (port 22821)
  │                             └─ AiProxyV1Servlet → AiProxyLlmRunnerImpl
  │                                  └─ tool-calling loop（search_knowledge_base / query_database）
  │    → 填入 md-preview div（Markdown 渲染，點擊可切回 textarea 編輯）
```

### ⚠️ Port 22821 = Tomcat，不是獨立 AiProxyServer

`start-cbm-lite.bat` 會啟動獨立 `AiProxyServer`（預設 port 4141，讀 `C:\com\proxy\aiproxy.properties`），
但 `cbm-lite.properties` 的 `cbm.lite.llm.url` 指向 `http://127.0.0.1:22821/aipower/v1/...`，
這是 **Tomcat（server.bat）** 處理的 endpoint，路徑含 `/aipower/` 前綴。

**修改 `AiProxyLlmRunnerImpl.java` 後，必須複製 class 到 Tomcat WEB-INF/classes 並重啟 Tomcat，
光重啟 cbm-lite 無效。**

---

## Java Endpoints（CbmLiteServer.java）

### 1. `/agent/mode` — 讀寫模式

| Method | 行為 |
|--------|------|
| GET    | 讀 `C:\com\cbm-lite\config\agent-mode.txt`，回 `{"mode":"ai"}` 或 `{"mode":"claude"}` |
| POST   | Body `{"mode":"ai"}` 或 `{"mode":"claude"}`，寫檔，回 `{"success":true,"mode":"..."}` |

mode 值：`"ai"` = AI 建議開啟；`"claude"` = 人工模式（建議不觸發）

### 2. `/agent/suggest` — AI 建議回覆

| Method | 行為 |
|--------|------|
| GET `?roomId=xxx` | 查 `TcChatMessage`（最近 30 則） → 組 LLM history → 呼叫 AiProxy → 回 `{"suggestion":"..."}` |

System prompt（寫死在 `agentSuggest` method）：
```
你是客服回覆助理，依序執行以下步驟：
1) 必須先呼叫 search_knowledge_base 搜尋知識庫。
2) 若知識庫有答案直接用，無需再查其他。
3) 若知識庫無完整答案，再用 query_database 查 CRM 補充。
最後只輸出回覆建議本身（2~4句繁體中文），不要加任何說明或前綴。
```

---

## AiProxy 工具整合（AiProxyLlmRunnerImpl.java）

### Tool-calling 模式判斷

```java
boolean   transparent  = clientSystem != null && !clientSystem.trim().isEmpty();
String    activeSystem = transparent ? clientSystem : ecpSystemPrompt;
JSONArray activeTools  = ecpTools;   // 永遠給全部工具（KB + CRM + PBX）
```

**透明模式（transparent=true）**：agentSuggest 帶了 system 訊息 → 使用 client 的 system prompt，
工具仍然全部開放（`search_knowledge_base` + `query_database`）。

### search_knowledge_base 工具實作

**⚠️ 坑：llmwiki search API 的 snippet 只截到 frontmatter metadata，實際內容讀不到。**

正確做法：用搜尋結果的 `path` 欄位，從磁碟讀完整 md 檔，剝掉 frontmatter 後送給 AI。

```java
private static String searchKnowledgeBase(String query) {
    String base = "http://127.0.0.1:19828";
    String body = new JSONObject().put("query", query).put("limit", 3).toString();
    String resp = AiProxyLlmClientImpl.httpPost(base + "/api/v1/projects/current/search", body, null, 10000);
    JSONArray raw = new JSONObject(resp).optJSONArray("results");
    if (raw == null || raw.length() == 0) return new JSONObject().put("results", "無相關知識庫內容").toString();

    // 取 wiki 根目錄（/api/v1/projects → currentProject.path）
    String wikiRoot = null;
    try {
        String proj = wikiGet(base + "/api/v1/projects");
        JSONObject cp = new JSONObject(proj).optJSONObject("currentProject");
        if (cp != null) wikiRoot = cp.optString("path", null);
    } catch (Exception ignored) {}

    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < raw.length(); i++) {
        JSONObject r = raw.getJSONObject(i);
        String content = null;
        if (wikiRoot != null) {
            try {
                // 讀完整檔案（不用截斷的 snippet）
                String filePath = wikiRoot + "/" + r.optString("path", "");
                content = new String(java.nio.file.Files.readAllBytes(
                    java.nio.file.Paths.get(filePath)), java.nio.charset.StandardCharsets.UTF_8);
                // 剝掉 frontmatter（--- ... \n---\n 之後才是正文）
                int end = content.indexOf("\n---\n", 3);
                if (end >= 0) content = content.substring(end + 5).trim();
            } catch (Exception ignored) { content = null; }
        }
        if (content == null) {   // fallback to snippet
            content = r.optString("snippet", "");
            int sep = content.indexOf("---", 3);
            if (sep >= 0) content = content.substring(sep + 3).trim();
        }
        String title = r.optString("title", "");
        if (!content.isEmpty()) sb.append("【").append(title).append("】\n").append(content).append("\n---\n");
    }
    return new JSONObject().put("results", sb.toString().trim()).toString();
}

private static String wikiGet(String url) throws Exception {
    java.net.HttpURLConnection c = (java.net.HttpURLConnection) new java.net.URL(url).openConnection();
    c.setConnectTimeout(4000); c.setReadTimeout(4000);
    try (java.io.InputStream is = c.getInputStream()) {
        return new String(is.readAllBytes(), java.nio.charset.StandardCharsets.UTF_8);
    }
}
```

llmwiki 資料路徑：wiki 根 = `/api/v1/projects` → `currentProject.path`（例：`D:/WIKI/HCH`）；
entity 檔在 `<root>/wiki/entities/<名稱>.md`。

---

## UI — Markdown 預覽

### 結構（一次只顯示一個框）

```html
<div class="composer">
  <div class="input-wrap">
    <!-- AI 填入後顯示 preview（Markdown 渲染），點擊切回 textarea 編輯 -->
    <div id="md-preview" class="md-preview"
         style="display:none;cursor:pointer"
         onclick="showEditor()" title="點擊編輯"></div>
    <!-- 空白或使用者手動輸入時顯示 textarea -->
    <textarea id="input"
              placeholder="輸入回覆內容，Enter 送出、Shift+Enter 換行"
              onblur="if(this.value.trim())showPreview()"></textarea>
  </div>
  ...
</div>
```

CSS：

```css
.input-wrap { flex:1; display:flex; flex-direction:column; gap:4px; }
.composer textarea {
  width:100%; box-sizing:border-box; resize:none; height:70px;
  border:1px solid #cfd8dc; border-radius:6px; padding:10px;
  font-size:14px; font-family:inherit; overflow-y:auto; scrollbar-width:none;
}
.composer textarea::-webkit-scrollbar { display:none; }
.md-preview {
  border:1px solid #cfd8dc; border-radius:6px; padding:10px;
  font-size:14px; min-height:40px; background:#f9fbff;
  line-height:1.5; word-break:break-all;
}
```

### JavaScript

```javascript
function renderMarkdown(s) {
  return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;')
    .replace(/\*\*(.*?)\*\*/g,'<strong>$1</strong>')
    .replace(/__(.*?)__/g,'<strong>$1</strong>')
    .replace(/\*(.*?)\*/g,'<em>$1</em>')
    .replace(/`([^`]+)`/g,'<code>$1</code>')
    .replace(/^#{1,6}\s+(.*)/gm,'<b>$1</b>')
    .replace(/\n/g,'<br>');
}

function showPreview() {
  var inp = document.getElementById('input');
  var pre = document.getElementById('md-preview');
  if (!inp || !pre) return;
  var txt = inp.value.trim();
  if (!txt) { showEditor(); return; }
  pre.innerHTML = renderMarkdown(txt);
  pre.style.display = 'block';
  inp.style.display = 'none';
}

function showEditor() {
  var inp = document.getElementById('input');
  var pre = document.getElementById('md-preview');
  if (pre) pre.style.display = 'none';
  if (inp) { inp.style.display = ''; inp.focus(); }
}

function updatePreview() { showPreview(); }

function clearInput() {
  var inp = document.getElementById('input');
  if (inp) inp.value = '';
  showEditor();
}
```

### checkAndSuggest / regenerate 填入後呼叫 showPreview

```javascript
function checkAndSuggest(msgs) {
  if (agentMode !== 'ai' || !msgs || !msgs.length || !cur) return;
  var last = msgs[msgs.length - 1];
  if (last.category !== 'Outer') return;
  if (last.id === lastSuggestedMsgId) return;
  lastSuggestedMsgId = last.id;
  fetch(CBM_BASE + '/agent/suggest?roomId=' + cur)
    .then(function(r) { return r.json(); })
    .then(function(d) {
      var inp = document.getElementById('input');
      if (inp && !inp.disabled && d.suggestion) { inp.value = d.suggestion; showPreview(); }
    }).catch(function() {});
}

function regenerate() {
  if (!cur) return;
  lastSuggestedMsgId = null;
  fetch(CBM_BASE + '/agent/suggest?roomId=' + cur)
    .then(function(r) { return r.json(); })
    .then(function(d) {
      var inp = document.getElementById('input');
      if (inp && !inp.disabled && d.suggestion) { inp.value = d.suggestion; showPreview(); }
    }).catch(function() {});
}
```

send() 成功後清空：`input.value = ''; showEditor();`

⚠️ **Java text block 中的 JS regex**：`\*` 要寫成 `\\*`，否則 javac 報 invalid escape。

---

## 編譯與重啟

### CbmLiteServer.java（UI / agentSuggest）

```powershell
& "C:\com\chainsea\jdk\bin\javac.exe" -encoding UTF-8 `
  -d "C:\com\cbm-lite\classes" `
  -cp "C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\*" `
  "C:\com\cbm-lite\src\CbmLiteServer.java"

$p = (Get-NetTCPConnection -LocalPort 12621 -ErrorAction SilentlyContinue | Select-Object -First 1).OwningProcess
if ($p) { Stop-Process -Id $p -Force }
Start-Sleep -Seconds 2
Start-Process "C:\com\cbm-lite\start-cbm-lite.bat"
Start-Sleep -Seconds 5
(Invoke-WebRequest "http://127.0.0.1:12621/health" -UseBasicParsing -TimeoutSec 5).StatusCode
```

### AiProxyLlmRunnerImpl.java（KB 搜尋 / tool loop）→ 須額外複製到 Tomcat

```powershell
# 1. 編譯（tool/src 全部一起編）
& "C:\com\chainsea\jdk\bin\javac.exe" -encoding UTF-8 `
  -d "C:\com\chainsea\tool\classes" `
  -cp "C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\*" `
  (Get-ChildItem "C:\com\chainsea\tool\src\*.java" | Select-Object -ExpandProperty FullName)

# 2. 複製到 Tomcat WEB-INF/classes
$src  = "C:\com\chainsea\tool\classes\com\chainsea\ecp\aiproxy"
$dest = "C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\classes\com\chainsea\ecp\aiproxy"
New-Item -ItemType Directory -Force $dest | Out-Null
Copy-Item "$src\AiProxyLlmRunnerImpl.class" $dest -Force
Copy-Item "$src\AiProxyLlmRunnerImpl`$*.class" $dest -Force -ErrorAction SilentlyContinue

# 3. 重啟 Tomcat（只殺 Tomcat，不動 cbm-lite 和 MariaDB）
$tomcatPid = (Get-NetTCPConnection -LocalPort 22821 -State Listen | Select-Object -First 1).OwningProcess
Stop-Process -Id $tomcatPid -Force
Start-Process "C:\com\chainsea\server.bat" -WindowStyle Hidden

# 4. 等啟動（約 35 秒）
until ($(powershell -Command "(Get-NetTCPConnection -LocalPort 22821 -State Listen -ErrorAction SilentlyContinue).Count -gt 0")) { Start-Sleep 3 }
(Invoke-WebRequest "http://127.0.0.1:22821/aipower/" -UseBasicParsing -TimeoutSec 5).StatusCode
```

---

## 已知坑

| 問題 | 原因 | 解法 |
|------|------|------|
| agentSuggest AI 說「查不到」但知識庫有資料 | llmwiki search API snippet 只截到 frontmatter，正文沒進去 | `searchKnowledgeBase()` 改為用 `path` 欄位讀磁碟完整 md 檔 |
| 改了 AiProxyLlmRunnerImpl 但行為沒變 | 只重啟 cbm-lite；Tomcat 用的是 WEB-INF/classes 的舊 class | 複製 .class 到 Tomcat WEB-INF/classes 並重啟 Tomcat |
| 輸入區變成上下兩個框 | preview div + textarea 同時顯示 | `showPreview()`/`showEditor()` 互斥，一次只顯示一個 |
| AI 選擇查 CRM 而不查知識庫 | tools 全開時 AI 自行決定，偏好 query_database | agentSuggest system prompt 明確三步驟：先 KB→無答案才 CRM |
| JSONArray.putAll() 不存在 | org.json 版本較舊 | 改用 for 迴圈逐一 put |
| header 高度切換時跳動 | 按鈕文字 AI/人工寬度不同 | `min-width:60px; line-height:1` 鎖住尺寸 |
| AI 模式建議重複觸發 | 每 2 秒輪詢 | `lastSuggestedMsgId` 對比最後一則 Outer 訊息 ID |
| agent-monitor.py 衝突 | Python 會自動送出，與新邏輯重複 | 確認 PID 已終止：`Get-Process python* \| Stop-Process` |

---

## Conformance Addendum

## When to Use
LINE 文字客服台 AI 建議回覆功能 — /agent/suggest、知識庫整合、Markdown 預覽、AiProxy tool-calling 架構，改動橫跨 CbmLiteServer.java 與 AiProxyLlmRunnerImpl.java

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
