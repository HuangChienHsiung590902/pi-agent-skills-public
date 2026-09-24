---
name: xai-store-false-proxy
description: 當 Grok/xAI 透過 OmniRoute/pi 出現 "Response is too large to store" 錯誤時，部署本機 HTTPS proxy 強制加上 store=false。也適用於 xAI API 預設 store=true 導致長回應被拒的任何場景。
---

# xAI store=false Proxy

## When to Use

- 使用 Grok（xAI）模型時出現 `"Response is too large to store. You can avoid this error by setting store to false in your request."`
- pi agent 選 `omni/xao/grok-*` 或 `omni/xai-oauth/grok-*` 系列模型時長回應被拒
- 任何透過 OmniRoute 呼叫 xAI API（`api.x.ai`）時需要強制 `store=false` 的場景

## 架構

```
pi agent
  → OmniRoute (localhost:20128)
    → api.x.ai (hosts redirect → 127.0.0.1:443)
      → Proxy (port 443, Node.js HTTPS)
        + store: false 注入
        → REAL_IP:443 (Cloudflare, 繞過 hosts 迴圈)
```

## 相依

- **Node.js** ≥ 18（已安裝）
- **OpenSSL**（用於產生自簽憑證，`winget install OpenSSL.OpenSSL`）
- **系統管理員權限**（hosts 修改 + port 443 綁定）
- **Windows**

## Procedure

### 1. 部署 Proxy 腳本

```powershell
$dest = "$env:TEMP\pi-work\xai-store-false-proxy"
mkdir -Force $dest
Copy-Item "D:\OB\skills\xai-store-false-proxy\scripts\*" $dest -Recurse
cd $dest
```

### 2. 產生 SSL 憑證

```powershell
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 3650 -nodes `
  -subj "/CN=api.x.ai" -addext "subjectAltName=DNS:api.x.ai"
```

### 3. 以系統管理員啟動

```powershell
.\start.ps1
```

腳本會自動：
- 匯入憑證到 Windows 信任區（`certutil -addstore -user Root cert.pem`）
- 加入 `127.0.0.1 api.x.ai` 到 hosts
- 啟動 proxy 監聽 port 443

### 4. 設定 OmniRoute 使用系統 CA

在系統環境變數加入 `NODE_OPTIONS=--use-system-ca`，重啟 OmniRoute。

或在 `D:\.system\npm\node_modules\omniroute\.env` 加上 `NODE_OPTIONS=--use-system-ca`（需確認 OmniRoute 的 dotenv 是否在 Node.js 原生 TLS 之前載入；若不生效，改用系統環境變數）。

### 5. 驗證

用 pi agent 選 Grok 模型問一個長問題，觀察 proxy 終端機輸出：

```
🔧 store→false POST /v1/chat/completions
```

若出現 `➡️` 而非 `🔧`，代表 xAI 已自帶 `store:false`，可考慮移除 proxy。

## 移除

```powershell
# 停止 proxy（關閉 admin PowerShell 視窗）

# 移除 hosts 記錄（手動編輯或 PowerShell admin）
# C:\Windows\System32\drivers\etc\hosts → 刪除 "127.0.0.1 api.x.ai"

# 移除憑證
certutil -delstore -user Root api.x.ai

# 刪除暫存
rm -r $env:TEMP\pi-work\xai-store-false-proxy
```

## Pitfalls

- **Cloudflare IP 會變動**：若 proxy 突然 timeout，需更新 `proxy.mjs` 中的 `REAL_IP`。取得最新 IP：
  ```powershell
  nslookup -type=A api.x.ai 1.1.1.1
  ```
- **Port 443 被佔用**：IIS、VMware、Skype 可能佔用 443。用 `netstat -ano | findstr :443` 確認
- **憑證過期**：自簽憑證有效期 10 年
- **NODE_OPTIONS 需重啟 OmniRoute**：修改後必須重啟才能生效
- **`content-length` 必須更新**：`proxy.mjs` 已處理，修改 body 後自動更新 header

## Verification

1. Proxy 終端機顯示 `🔧 store→false POST /v1/chat/completions`
2. Grok 長回應不再出現 `"Response is too large to store"`
3. `curl -k https://127.0.0.1:443/v1/models` 回傳 xAI 的 `{"code":"unauthenticated:..."}`
4. `Get-NetTCPConnection -LocalPort 443` 顯示 `Listen` 狀態

## Inputs and Outputs

### Inputs

- xAI/OmniRoute 的實際錯誤、proxy 設定、TLS 憑證與本機 port 狀態。
- 需要通過 proxy 驗證的 `/v1/models` 或 chat/completions 請求。

### Outputs

- 只在需要時插入 `store=false` 的本機 HTTPS proxy，以及 upstream/health 驗證結果。
- 變更前備份、啟停命令、監聽 port 與未解決問題。

## Rules and Limitations

- 不要在 Skill、日誌或回報中揭露 xAI API key、OAuth token 或 TLS 私鑰。
- 只能修改已確認的本機 proxy 設定；不要把服務直接暴露到公網或擅自更換 production domain。
- 只有實際請求與回應都經過 proxy，才能宣稱 `store=false` 已生效；`/v1/models` 的錯誤不能單獨證明 chat 已修復。
- 修改服務前先保留設定備份，完成後驗證 upstream、proxy、TLS 與 listening port。
