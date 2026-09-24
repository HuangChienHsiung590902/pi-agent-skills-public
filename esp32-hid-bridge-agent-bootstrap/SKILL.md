---
name: esp32-hid-bridge-agent-bootstrap
description: 舊名稱相容入口；WindowsBridgeAgent 的下載、版本管理、工作排程常駐、Cloudflare WSS、截圖/OCR 與 MCP 工具流程已整併到 esp32-hid-bridge-windows-agent。
description_zh: ESP32 HID Bridge Agent bootstrap 相容轉介。
---

# ESP32 HID Bridge Agent Bootstrap（相容入口）

## When to Use

當舊索引、舊對話或使用者提到 WindowsBridgeAgent bootstrap、登入自啟、Agent 下載與重連時使用。

## Procedure

1. 立即讀取：
   ```text
   D:\OB\skills\esp32-hid-bridge-windows-agent\SKILL.md
   ```
2. 依新 skill 的安全版本管理、GitHub Release、Cloudflare WSS、互動使用者工作排程、截圖/OCR 與實機驗證流程執行。
3. 不再使用本檔過去記載的手機 `5590` 下載、硬編碼手機 IP、舊 `screen_ocr.ps1` 或 `0.2.0` minimal agent 做法。

## Pitfalls

- 本檔只是避免舊名稱失效，不是另一套實作。
- 不要把舊 bootstrap 與新流程同時套用。
- 公開 EXE 不可內嵌 token、手機 IP 或預設 WebSocket URL。

## Verification

確認實際操作依據是：

```text
D:\OB\skills\esp32-hid-bridge-windows-agent\SKILL.md
```

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
