---
name: mikopbx
description: Mikopbx (Asterisk-based PBX) administration, inspection, and troubleshooting via web UI and API. Use when managing extensions, viewing CDR/call records, inspecting SIP peer settings, checking system logs, or diagnosing call drops in a Mikopbx system. For the `mikopbx2026-docker` container's own Docker/network/port config, use that skill instead; for building an AI voice bot integration on top of it (Asterisk ARI + External Media), see `voice-bot-mikopbx-ari`.
---

# Mikopbx 管理與故障排除

> 這是通用 PBX 管理/查詢（讀取 CDR、擴展設定、系統日誌）。若是要在 `mikopbx2026-docker` 這個容器上做 AI 語音 bot 整合（真人撥打分機由 AI 接聽），改用 `voice-bot-mikopbx-ari` skill；該容器本身的 Docker 管理見 `mikopbx2026-docker` skill。

## 快速參考

| 項目 | 値 |
|------|-----|
| 登入網址 | `http://{host}:8080` |
| 登入帳號 | `admin` |
| 預設密碼 | `admin` 或系統管理者提供 |
| Session 起始 | `POST /admin-cabinet/session/start` |
| AMI 連接埠 | `5038` (需防火牆例外) |
| SSH 連接埠 | `23` |

## 認證登入（Python + requests）

```python
import urllib.request, urllib.parse, http.cookiejar

cj = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cj))
opener.addheaders = [('User-Agent', 'Mozilla/5.0')]

# 1. 取得登入頁面
opener.open("http://10.145.119.64:8080")

# 2. POST 登入
login_data = urllib.parse.urlencode({
    'login': 'admin',
    'password': '<ECP_PASSWORD>',
    'WebAdminLanguage': 'en',
    'rememberMeCheckBox': 'on'
}).encode('utf-8')
opener.open("http://10.145.119.64:8080/admin-cabinet/session/start",
            data=login_data, timeout=8)
```

## Playwright 瀏覽器自動化

Mikopbx 是 SPA（所有路由都回到同一個 HTML），**必須用瀏覽器執行 JavaScript** 才能讀取動態內容。

### 基本登入流程

```python
from playwright.sync_api import sync_playwright

BASE = "http://10.145.119.64:8080"

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context(viewport={"width": 1920, "height": 1080})
    page = context.new_page()
    page.set_default_timeout(20000)

    def goto(url, wait=5000):
        full = url if url.startswith("http") else BASE + url
        page.goto(full)
        page.wait_for_load_state("domcontentloaded")
        page.wait_for_timeout(wait)

    # 登入
    goto("/")
    page.fill('input[name="login"]', 'admin')
    page.fill('input[name="password"]', '<ECP_PASSWORD>')
    page.press('input[name="password"]', 'Enter')
    goto("/admin-cabinet/", wait=6000)  # 等待 SPA 渲染
```

### 關鍵頁面 URL

| 功能 | URL |
|------|-----|
| 擴展清單 | `/admin-cabinet/extensions/index/` |
| 擴展設定（ID 查表得知） | `/admin-cabinet/extensions/modify/{id}` |
| 通話詳情紀錄 (CDR) | `/admin-cabinet/call-detail-records/index/` |
| 系統診斷/日誌 | `/admin-cabinet/system-diagnostic/index/` |
| SSH Console | `/admin-cabinet/console/index/` |
| 一般設定 | `/admin-cabinet/general-settings/modify/` |
| 來電路由 | `/admin-cabinet/incoming-routes/index/` |
| 電信供應商 | `/admin-cabinet/providers/index/` |
| 外部路由 | `/admin-cabinet/outbound-routes/index/` |
| 韌體更新 | `/admin-cabinet/update/index/` |

### 讀取擴展設定

```python
# 從擴展清單抓 ID
goto("/admin-cabinet/extensions/index/", 5000)
rows = page.query_selector_all("table tbody tr")
ext_map = {}
for row in rows:
    cells = row.query_selector_all("td")
    if len(cells) >= 3:
        num = cells[2].inner_text().strip()
        edit_link = row.query_selector("a[href*='/modify/']")
        href = edit_link.get_attribute("href") if edit_link else ""
        if num.isdigit() and href and not href.startswith("http"):
            ext_map[num] = href

# 讀取特定擴展設定
if '101' in ext_map:
    goto(ext_map['101'], 6000)
    inputs = page.query_selector_all("input, select")
    for inp in inputs:
        name = inp.get_attribute("name") or ""
        typ = inp.get_attribute("type") or ""
        try:
            if typ in ["text", "number", "email", "password", "hidden"]:
                val = inp.input_value()
                if name and val and "uniqid" not in name:
                    print(f"  {name} = {val}")
            elif typ == "checkbox" and inp.is_checked():
                print(f"  {name} = true")
        except:
            pass
```

### 讀取 CDR 通話記錄

```python
goto("/admin-cabinet/call-detail-records/index/", 6000)
rows = page.query_selector_all("table tbody tr")
for row in rows:
    cells = row.query_selector_all("td")
    cell_texts = [c.inner_text().strip() for c in cells]
    print("  ", cell_texts)
```

### 系統診斷日誌

```python
goto("/admin-cabinet/system-diagnostic/index/", 6000)
# 點擊 "Show log" 按鈕
show_log = page.get_by_text("Show log")
show_log.click()
page.wait_for_timeout(5000)
# 找關鍵字
body = page.inner_text("body")
for line in body.split('\n'):
    if any(k in line for k in ['BYE', 'HANGUP', '102', '103', 'ERROR']):
        print(line[:150])
```

## 常見故障排除

### CDR 顯示空 duration（通話中斷）

空 duration 代表通話異常中斷，不是正常結束。檢查：
1. System Diagnostic 日誌中是否有 `BYE`、`CANCEL`、`timeout`
2. 擴展的 `fwd_ringlength` 設定（預設 45 秒）
3. 來電路由規則

### BYE 掛斷調查

BYE 來源判斷：
- `From: <sip:102@{pbx}>` → 分機 102 發起掛斷（常是設備行為）
- `Via: 8.8.8.8:5060` → BYE 經轉發，源頭可能是外部設備/NAT
- `User-Agent: mikopbx-*` → 但 Via 非本機，說明是轉發的 SIP 訊息

### Extension 狀態 "Offline"

擴展未註冊。檢查：
1. SIP 帳號密碼是否正確
2. 網路是否可達 PBX:5060
3. 防火牆是否阻擋

### 快速取得所有擴展 ID

```python
goto("/admin-cabinet/extensions/index/", 5000)
for row in page.query_selector_all("table tbody tr"):
    cells = row.query_selector_all("td")
    if len(cells) >= 3:
        num = cells[2].inner_text().strip()
        edit = row.query_selector("a[href*='/modify/']")
        href = edit.get_attribute("href") if edit else ""
        if num.isdigit():
            print(f"Ext {num} -> {href}")
```

## 腳本工具

到位於 `scripts/` 目錄下：

| 腳本 | 功能 |
|------|------|
| `scripts/login_inspect.py` | 登入並列出所有擴展 ID、CDR 記錄、系統設定 |
| `scripts/cdr_pull.py` | 提取 CDR 通話記錄（指定日期範圍） |

### scripts/login_inspect.py

```bash
python D:/Config/skills/mikopbx/scripts/login_inspect.py \
    --host 10.145.119.64 --port 8080 \
    --username admin --password <ECP_PASSWORD> \
    --check extensions,cdr,settings
```

輸出：擴展清單、CDR 記錄、一般設定。

## 注意事項

- Mikopbx UI 為 SPA，所有路徑皆為 JavaScript 動態渲染，Python requests 無法取得動態內容
- SSH Console 也是 JS 渲染的 xterm，無法用 requests 操作，需透過 AMI 或 SSH
- AMI (5038) 預設啟用但可能被防火牆阻擋
- `fwd_ringlength=45` 是分機鈴聲超時，非通話總時長

---

## Conformance Addendum

## When to Use
Mikopbx (Asterisk-based PBX) administration, inspection, and troubleshooting via web UI and API. Use when managing extensions, viewing CDR/call records, inspecting SIP peer settings, checking system logs, or diagnosing call drops in a Mikopbx system. For the `mikopbx2026-docker` container's own Docker/network/port config, use that skill instead; for building an AI voice bot integration on top of it (Asterisk ARI + External Media), see `voice-bot-mikopbx-ari`.

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

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
