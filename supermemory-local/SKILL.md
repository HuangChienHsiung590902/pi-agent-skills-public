---
name: supermemory-local
description: 在這台 Windows 電腦上安裝、啟動、檢查、除錯與驗證本機 Supermemory Local；當使用者提到 Supermemory、persistent memory、記憶服務、localhost:6767、OmniRoute 接線、寫入／搜尋記憶、服務無法啟動或要確認 Supermemory 是否可用時使用。涵蓋官方 Windows binary 安裝、OmniRoute OpenAI-compatible provider、Windows 登入自動啟動、非同步文件處理與 smoke test。不要把 API token 寫入 Skill 或回覆中。
compatibility: Windows；官方 server binary 位於 C:\\Users\\Administrator\\.supermemory\\bin\\supermemory-server.exe；OmniRoute 預期監聽 http://127.0.0.1:20128/v1；Supermemory 預期監聽 http://localhost:6767。
---

# Supermemory Local（Windows）

## When to Use

使用者要安裝或管理 Supermemory Local、把它接到 OmniRoute／OpenAI-compatible LLM、測試記憶寫入與搜尋、查詢本機服務狀態、修復啟動失敗，或詢問「Supermemory 能不能用了」時使用。

## Inputs and Outputs

### Inputs

- Supermemory 原始碼（可選）：`C:\Users\Administrator\supermemory`
- Server binary：`C:\Users\Administrator\.supermemory\bin\supermemory-server.exe`
- 本機資料目錄：`C:\Users\Administrator\.supermemory\data`
- OmniRoute endpoint：`http://127.0.0.1:20128/v1`
- LLM API token：只從既有的本機啟動腳本／安全環境取得，禁止要求使用者貼到對話，也禁止寫入本 Skill。

### Outputs

- Supermemory HTTP service：`http://localhost:6767`
- 可儲存、非同步處理並搜尋記憶的本機服務
- 驗證結果必須包含：服務 HTTP 狀態、文件狀態、是否成功產生／取回記憶，以及仍存在的限制。

## Known Installation

目前這台機器的已驗證安裝：

- 官方 GitHub Release `server-v0.0.8`
- Windows x64 binary：`C:\Users\Administrator\.supermemory\bin\supermemory-server.exe`
- `supermemory-server.exe --version` 可能顯示 bundled Bun runtime 版本 `1.3.4`，不要把它誤認為 Supermemory server release 版本。
- 官方 `npm supermemory local install` 在 Git Bash 可能把 `MINGW64_NT-*` 判斷成 unsupported OS；此情況改用官方 Windows x64 Release binary，並以 GitHub Release 的 `.sha256` 驗證。
- 使用者 PATH 已加入：`C:\Users\Administrator\.supermemory\bin`
- Windows 工作排程名稱：`Supermemory Server`
- 啟動腳本：`C:\Users\Administrator\.supermemory\start-supermemory.ps1`

## Provider Configuration

Supermemory Local 需要至少一個 LLM provider。OmniRoute 是 OpenAI-compatible endpoint，因此使用：

```powershell
$env:SUPERMEMORY_DATA_DIR = 'C:\Users\Administrator\.supermemory\data'
$env:OPENAI_BASE_URL = 'http://127.0.0.1:20128/v1'
$env:OPENAI_API_KEY = '<從安全的本機設定取得，不要貼出>'
$env:OPENAI_MODEL = 'auto/best-fast'
$env:OPENAI_FAST_MODEL = 'auto/best-fast'
$env:OPENAI_TEXT_MODEL = 'auto/best-fast'
& 'C:\Users\Administrator\.supermemory\bin\supermemory-server.exe'
```

啟動或修改 provider 前：

1. 先確認 OmniRoute 正常；可使用 OmniRoute status tool，或對 `http://127.0.0.1:20128/v1/models` 發送帶有本機 token 的請求。輸出只顯示 model 數量／名稱，不要顯示 token。
2. 若使用既有 `start-supermemory.ps1`，優先重用它，不要另建一份會造成 token 分散的設定。
3. 若 token 曾貼在聊天、shell history 或 log，提醒使用者撤銷並重建 token；不要把舊 token 複製到 Skill。
4. 本機 embedding 預設是 `Xenova/bge-base-en-v1.5`，不需要額外 embedding API key。
5. `OPENAI_MODEL` 使用 OmniRoute `/v1/models` 實際存在的 model；`auto/best-fast` 已驗證可完成基本 chat request。

## Procedure

### 1. 檢查 binary、OmniRoute 與 port

```powershell
Test-Path 'C:\Users\Administrator\.supermemory\bin\supermemory-server.exe'
Get-Process -Name supermemory-server -ErrorAction SilentlyContinue
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
  Where-Object { $_.LocalPort -in @(20128, 6767) }
```

若 OmniRoute 未在 `20128` 監聽，先修復 OmniRoute，不要啟動一個沒有 provider 的 Supermemory。

### 2. 啟動或重啟

優先執行既有啟動腳本：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File 'C:\Users\Administrator\.supermemory\start-supermemory.ps1'
```

要背景啟動可使用：

```powershell
Start-Process powershell.exe -WindowStyle Hidden -ArgumentList `
  '-NoProfile','-ExecutionPolicy','Bypass','-File',
  'C:\Users\Administrator\.supermemory\start-supermemory.ps1'
```

重新啟動前先停止同名 process，避免資料目錄 `.instance.lock` 造成誤判：

```powershell
Get-Process -Name supermemory-server -ErrorAction SilentlyContinue |
  Stop-Process -Force
```

### 3. 驗證服務首頁

```powershell
Invoke-WebRequest http://127.0.0.1:6767/ -UseBasicParsing
```

預期 HTTP `200` 且頁面包含 Supermemory。不要假設 `/health` 一定存在；已驗證版本對 `/health` 回傳 `404` 不代表服務故障。

### 4. 寫入測試記憶

每次 smoke test 使用唯一 marker 與隔離的 `containerTag`：

```python
import json, urllib.request, uuid
marker = "smoke-test-" + uuid.uuid4().hex[:10]
payload = {
    "content": f"Supermemory smoke test marker {marker}.",
    "containerTag": "smoke-test"
}
req = urllib.request.Request(
    "http://127.0.0.1:6767/v3/documents",
    data=json.dumps(payload).encode(),
    headers={"Content-Type": "application/json"},
)
print(urllib.request.urlopen(req, timeout=60).read().decode())
```

預期回應包含：

```json
{"id":"...","status":"queued"}
```

`queued` 只代表已接受，不代表已可搜尋。

### 5. 等待非同步處理並搜尋

先以回應的 `id` 輪詢：

```text
GET http://127.0.0.1:6767/v3/documents/<id>
```

狀態流程通常是 `queued → extracting/chunking/embedding → done`。只有到 `done` 才測試搜尋：

```json
POST http://127.0.0.1:6767/v3/search
{
  "q": "<測試 marker>",
  "containerTag": "smoke-test"
}
```

若文件已 `done` 但搜尋無結果，檢查：

- 搜尋使用的 `containerTag` 是否完全相同。
- 是否過早搜尋。
- 本機 embedding 是否已完成預熱。
- OmniRoute chat request 是否成功。
- server log 是否出現 provider、queue 或 ingestion error。

### 6. 檢查服務自動啟動

```powershell
schtasks.exe /Query /TN 'Supermemory Server' /FO LIST
```

工作排程應存在，啟動程式應指向：

```text
powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File C:\Users\Administrator\.supermemory\start-supermemory.ps1
```

## Rules and Limitations

- 永遠不要在 Skill、回覆、測試輸出或 log 中寫出完整 API token；用 `<redacted>`。
- 不要把 Supermemory 的本機 API key 與 OmniRoute 的 provider token 混淆；前者由 server 首次啟動產生，後者只供 LLM provider 使用。
- 不要因為 `POST /v3/documents` 回傳 `queued` 就宣稱測試失敗；必須輪詢至 `done` 或明確 `failed`。
- 不要直接刪除 `C:\Users\Administrator\.supermemory\data`；這會破壞既有記憶。若需重建，先取得使用者明確授權並備份。
- 預設服務是本機服務；不要未經使用者要求就將 `6767` 暴露到公網或修改 Windows Firewall。
- Supermemory Local 的 lite 版有官方文件所述的文件數限制；若接近上限，先告知，不要自行繞過授權。
- 純文字記憶、抽取、搜尋可使用 OmniRoute；圖片、影片及高階 PDF 理解可能需要 Gemini／Vertex provider，不能保證 OmniRoute 的一般文字模型支援這些能力。

## Troubleshooting

### `No model provider API key configured`

確認啟動腳本或目前 process 有設定 `OPENAI_BASE_URL` 與非空的 `OPENAI_API_KEY`。不要只在一個 Git Bash session 設定後，期待 Windows 工作排程自動繼承該環境。

### 啟動後 process 存在但 6767 沒有 listener

- 讀取前景啟動輸出或獨立 log。
- 確認沒有另一個 process 持有 `.instance.lock`。
- 確認 `SUPERMEMORY_DATA_DIR` 指向固定的 `C:\Users\Administrator\.supermemory\data`；不同工作目錄會建立不同資料庫。
- 確認 OmniRoute 可連線並且 model id 有效。

### 搜尋一直是空陣列

先查 `GET /v3/documents/<id>`。若仍是 `queued` 或 `processing`，繼續等待；若是 `done`，重新確認 marker 與 `containerTag`，再檢查 embedding 與 server log。

### 官方 npm installer 回報 unsupported OS

在 Git Bash 看到 `unsupported OS: MINGW64_NT-*` 時，不要反覆重試 installer。改用官方 GitHub Release 的 `supermemory-server-windows-x64.exe`，下載對應 `.sha256` 並驗證後放到既定 binary 路徑。

## Verification

完成任何安裝、設定或修復後，至少記錄：

1. binary 存在且版本／Release 可識別。
2. OmniRoute endpoint 可回應 `/v1/models`，token 已遮蔽。
3. Supermemory `6767` 有 listener，首頁 HTTP `200`。
4. `POST /v3/documents` 成功回傳 `queued`。
5. `GET /v3/documents/<id>` 最終為 `done` 或清楚記錄 `failed`。
6. 使用相同 `containerTag` 的搜尋能取回測試 marker 或由 API 明確回報可解釋的限制。
7. 若有修改自動啟動，`schtasks /Query` 能看到 `Supermemory Server`。

回覆使用者時簡潔報告：服務網址、provider 是否可用、寫入／處理／搜尋結果、資料目錄，以及任何未解決限制；絕不回傳秘密。

## Pitfalls

- 把 `queued` 當成已索引，導致錯誤判定搜尋壞掉。
- 從不同工作目錄啟動，讓資料被寫到另一個 `./.supermemory`。
- 只檢查 process，不檢查 `6767` listener。
- 使用 `/health` 當唯一健康檢查；目前版本可能回 `404`。
- 把 OmniRoute 的 model list 當成所有 model 都可用；先用小型 chat request 驗證實際 model。
- 將 API token 寫進 `SKILL.md`、Git、測試輸出、PowerShell transcript 或回覆。
- 為了修復啟動問題直接刪除 data directory，造成既有記憶遺失。
