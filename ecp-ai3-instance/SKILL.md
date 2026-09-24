---
name: ecp-ai3-instance
description: C:\ECP 是用官方安裝包(D:\ECP\AI3_Version\ecp-windows-8.5.03.02-20250409-105336 + VideoPage_8.5.03.01)全新安裝的第三套 aipower/ECP 實例，port 12821/12822，專門用來開發/測試官方 VideoPage 模組改 LiveKit。含啟動方式(server.bat 已整合 LiveKit+Caddy)、與 Lab2/com 兩套舊實例的關係、登入資訊。用在：這套實例需要重啟、想搞懂它跟其他 ECP 實例的差異、或要繼續在這套環境上開發視訊功能。
---

# C:\ECP — AI3_Version 全新安裝實例

## 這是第幾套 ECP？

目前機器上先後出現過三套獨立的 aipower/ECP 安裝，**互不相通、各自獨立的 DB**：

| 實例 | 路徑 | Port | 狀態 | 用途 |
|---|---|---|---|---|
| 舊 | `C:\com\chainsea` | 22821 | **已不存在**(2026-07-11 확認目錄已消失) | 原本的視訊客服(LiveKit)自建功能誕生地，見 `ecp-video-livekit` skill |
| Lab2 | `C:\Lab2\chainsea` | 22821 | 見 `ecp-lab2-instance` skill | 對外 Cloudflare Tunnel 服務的正式環境 |
| **這套(AI3)** | `C:\ECP`(來源安裝包在 `D:\ECP\AI3_Version\`) | **12821**(HTTP) / **12822**(HTTPS) | 2026-07-11 新裝，開發沙盒 | 拿官方 VideoPage 模組(OpenVidu)改寫成 LiveKit 版的開發/測試環境 |

**不要把這三套搞混。** 尤其 `ecp-video-livekit` skill 講的是「舊 C:\com\chainsea」那一套自建的 JoinVideoServlet+Cloudflare Tunnel 架構，跟這裡的官方 VideoPage 模組+獨立 git repo+Caddy 架構是兩回事，細節見 `ecp-ai3-video-livekit` skill。

## 登入資訊

`http://127.0.0.1:12821/ecp` 或 `https://127.0.0.1:12822/ecp`(自簽憑證)
帳號：`administrator` / `3goY~1-A`(安裝時隨機產生，**建議盡快改密碼**，做法見 `ecp-pwd` skill)

## 一鍵啟動(2026-07-11 已把 LiveKit + Caddy 併入 server.bat)

```
C:\ECP\server.bat
```

這支 bat 現在依序做：
1. 背景啟動 `livekit\livekit-server.exe --config config.yaml`(訊令 7880、TCP 媒體 7881、UDP 媒體 7882)
2. 背景啟動 `caddy\caddy.exe run --config Caddyfile`(監聽 `10.145.119.100:7443`，TLS terminate 後 reverse_proxy 到本機 `127.0.0.1:7880`，給手機瀏覽器用——鏡頭/麥克風需要 secure context)
3. 前景執行 `apache-tomcat/bin/catalina run`(Tomcat，這行會佔住視窗，關掉視窗=關掉 Tomcat，但 LiveKit/Caddy 是背景行程不會跟著關)

Tomcat 用內嵌 MariaDB(資料在 `C:\ECP\mariadb\data`)，`server.bat` 啟動時會自己帶起來，不需要另外跑 `mariadb\database.bat`(那支是獨立維護模式用的，兩個不能同時跑，會鎖 data 目錄，細節同 `ecp-server-startup` skill)。

## 檢查有沒有正常起來

```powershell
tasklist /FI "IMAGENAME eq java.exe"
tasklist /FI "IMAGENAME eq livekit-server.exe"
tasklist /FI "IMAGENAME eq caddy.exe"
netstat -ano | findstr ":12821 :12822 :7880 :7443"
```

三個行程、四個對外相關 port(12821/12822/7880/7443)都要有才算完整起來。**只有 java.exe 沒有 livekit/caddy 也能連上 ECP 主畫面**，但視訊交談會卡在訊令連得上、媒體連不上(跟 `ecp-video-livekit` skill 記錄的舊坑同一個症狀)。

## 目前只是開發沙盒，不是對外服務

Config.js(前端視訊設定)目前寫死指向實體區網 IP `192.168.1.106`(2026-07-11 從 ZeroTier 虛擬網卡 IP
`10.145.119.100` 改過來，兩者都能連但區網延遲更低更穩，細節見 `ecp-ai3-video-livekit` skill 踩坑 8)，
**沒有經過 Cloudflare Tunnel**，只能區網內測試(手機需跟這台機器同一個 Wi-Fi，或跟 ZeroTier 一樣加入同一個
虛擬網路)。要對外開放的話需要仿照 Lab2 的做法另外接 Tunnel 網域，目前(2026-07-11)還沒做。

## 相關 skill

`ecp-ai3-video-livekit`(這套實例上的 VideoPage LiveKit 改寫細節與踩坑)、`ecp-lab2-instance`(對照另一套正式環境)、`ecp-video-livekit`(舊實例的自建版本，架構不同)、`ecp-server-startup`(mariaDB4j 相關陷阱)、`chainsea-feature-migration`(這次改寫用的方法論起點)。

---

## Conformance Addendum

## When to Use
C:\ECP 是用官方安裝包(D:\ECP\AI3_Version\ecp-windows-8.5.03.02-20250409-105336 + VideoPage_8.5.03.01)全新安裝的第三套 aipower/ECP 實例，port 12821/12822，專門用來開發/測試官方 VideoPage 模組改 LiveKit。含啟動方式(server.bat 已整合 LiveKit+Caddy)、與 Lab2/com 兩套舊實例的關係、登入資訊。用在：這套實例需要重啟、想搞懂它跟其他 ECP 實例的差異、或要繼續在這套環境上開發視訊功能。

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
