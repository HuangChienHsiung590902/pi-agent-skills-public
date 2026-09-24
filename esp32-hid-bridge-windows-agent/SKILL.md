---
name: esp32-hid-bridge-windows-agent
description: 維護 D:\Github\esp32_hid_bridge、WindowsBridgeAgent、Cloudflare WSS relay、公開 Release、Windows 截圖/OCR、MCP 工具命名與 SFAA 常駐部署；適用於 Agent 離線、版本不一致、OCR/截圖失敗、連錯電腦或 HID 部署問題。
description_zh: ESP32 HID Bridge 與 Windows Agent 的完整開發、發布、部署及除錯流程。
---

# ESP32 HID Bridge Windows Agent

## When to Use

使用者提到以下任務時使用本 skill：

- 正確手機 App `D:\Github\esp32_hid_bridge` 的開發、建置或部署。
- `WindowsBridgeAgent.exe` 的版本、GitHub Release、下載、常駐、截圖或 OCR。
- `mcp.james-huang.org` 的 MCP 或 `/bridge/ws` Agent 回連。
- MCP 的 `windows_*`、`hid_*` 工具失敗或工具名稱重複。
- GRAM、SFAA 同時連線後操作到錯誤電腦。
- ESP32-S3 HID 中文輸入法干擾、重複字母、HID 部署命令中的 Base64 損壞，或 COM port 韌體燒錄。

## Authoritative Paths

```text
正確手機 App：D:\Github\esp32_hid_bridge
Android package：com.esp32bridge.esp32_hid_bridge
Agent 原始碼：D:\Github\esp32_hid_bridge\windows_bridge_agent\Program.cs
Agent project：D:\Github\esp32_hid_bridge\windows_bridge_agent\WindowsBridgeAgent.csproj
App 主程式：D:\Github\esp32_hid_bridge\lib\main.dart
背景服務/MCP relay：D:\Github\esp32_hid_bridge\lib\background_service.dart
Agent bundled asset：D:\Github\esp32_hid_bridge\assets\windows_bridge\WindowsBridgeAgent.exe
ESP32 韌體：D:\Github\esp32_s3_ble_firmware\esp32_s3_ble_firmware.ino
```

`D:\Github\esp32_hid_bridge` 是含「用 HID 部署」的正確 App，不可誤稱為參考或錯誤 App。`esp32_s3_ble_controller` 是另一套精簡 BLE controller。

## Current Architecture

```text
GitHub Public Release
  └─ 下載 WindowsBridgeAgent.exe

WindowsBridgeAgent
  └─ wss://mcp.james-huang.org/bridge/ws
       └─ Cloudflare Tunnel hch
            └─ 手機 127.0.0.1:5588/bridge/ws

ChatGPT MCP
  └─ https://mcp.james-huang.org/mcp/<runtime-token>
       └─ 手機 127.0.0.1:5588/mcp/...
```

- Cloudflare Tunnel：`hch`
- Tunnel ID：`fca39237-d676-470f-ad9b-4fe870d94026`
- 手機 Cloudflare origin：`http://127.0.0.1:5588`
- `mcp.james-huang.org` 同時承載 MCP 與 Agent WebSocket。
- Agent URL 必須是 `wss://mcp.james-huang.org/bridge/ws`，不是 `hid.james-huang.org`。
- GitHub 只供公開下載 Agent；被控端不必安裝 ZeroTier。
- ZeroTier 可繼續供手機管理使用，但 Agent 不可依賴 `192.168.100.*` 或手機 Wi-Fi IP。
- Cloudflare 在 Termux 由 `runsv` 常駐，使用 `--protocol http2`。

## Security Rules

公開 EXE 不可內嵌：

- Relay token
- 手機 IP
- 預設 WebSocket URL

必須在執行時提供：

```powershell
WindowsBridgeAgent.exe --phone <ws-or-wss-url> --token <token>
```

無參數時應以 exit code `2` 結束。不要將實際 token 寫入 skill、Git repository、Release 說明或公開日誌。

對外 Agent 若擴充工具，預設 read-only；開放命令執行、寫檔等能力時，必須限制 token、目標電腦與使用範圍。

## Current Release

```text
Version：0.2.3
Repository：https://github.com/HuangChienHsiung590902/windows-bridge-agent-releases
Release：https://github.com/HuangChienHsiung590902/windows-bridge-agent-releases/releases/tag/v0.2.3
Download：https://github.com/HuangChienHsiung590902/windows-bridge-agent-releases/releases/download/v0.2.3/WindowsBridgeAgent.exe
Size：67,999,323 bytes
SHA-256：67714AB67EF927734383ED4C29C99921D9F03EEF9B2D547DC6659E984E4E8AA7
```

App 部署時先執行本機 EXE `--version`。版本已相同就直接啟動，不重複下載；不同才從 GitHub Release 下載。

## MCP Tool Naming

只保留一個清楚名稱，不要再建立 `agent_*`、`win_*` 同義別名：

| 工具 | 使用者意義 |
|---|---|
| `windows_status` | 查看被控 Windows 是否在線、電腦名稱與 Agent 版本 |
| `windows_install_agent` | 透過 ESP32 HID 安裝並啟動 Agent |
| `windows_screenshot` | 取得被控 Windows 螢幕圖片 |
| `windows_read_screen` | 讀取螢幕文字及座標；內部實作是 OCR |
| `windows_run_command` | 執行 PowerShell/CMD 並回傳結果 |
| `windows_read_file` | 讀取 UTF-8 文字檔 |
| `windows_write_file` | 寫入 UTF-8 文字檔 |

`windows_read_screen` 的意義是「讀取 Windows 螢幕上的文字」，不必向使用者暴露 OCR 術語。工具描述以繁體中文寫清楚用途。

HID 工具目前保留 `hid_type`、`hid_key`、`hid_move`、`hid_click`、`hid_scroll`、`hid_open_command_line`、`hid_type_command`、`hid_run_command`、`hid_batch`、`hid_status`。

## Procedure

以下依任務選用對應子流程；所有修改都必須完成該節的實機驗證，不可只以 build 或連線成功結案。

### Agent Build and Release

1. 修改前讀取 `Program.cs` 與 `.csproj`，確認版本及 target framework。
2. Agent GUI/螢幕功能使用：
   ```xml
   <TargetFramework>net8.0-windows</TargetFramework>
   <UseWindowsForms>true</UseWindowsForms>
   ```
3. 每次 binary 行為有變更就升版，不可讓原始碼顯示舊版本但 Release binary 不一致。
4. 發布 win-x64 self-contained single-file，例如：
   ```powershell
   dotnet publish -c Release -r win-x64 --self-contained true /p:PublishSingleFile=true
   ```
5. 驗證：
   ```powershell
   WindowsBridgeAgent.exe --version
   Get-Item WindowsBridgeAgent.exe | Select Length
   Get-FileHash WindowsBridgeAgent.exe -Algorithm SHA256
   ```
6. 掃描原始碼與 EXE 周邊設定，確認沒有 token、手機 IP 或內嵌 Relay URL。
7. 上傳新的 Public GitHub Release。
8. 以匿名方式從 GitHub 重新下載，驗證版本、大小及 SHA-256。
9. 更新 `main.dart`、`background_service.dart` 的版本與下載 URL。
10. Build APK、ADB 安裝、重新啟動 App，再透過 MCP 實測。

## Windows Persistent Deployment

不要從短暫 SSH Session 0 直接啟動 Agent後就認定能常駐；SSH 結束時程序可能被終止。

SFAA 已驗證的模式：

```text
Host：192.168.100.144（僅目前 LAN 管理位址，不可寫成 Agent 架構依賴）
Hostname：SFAA
互動使用者：SFAA\CS
Agent：C:\Users\CS\AppData\Local\Temp\wba.exe
Scheduled Task：WindowsBridgeAgent
Session：互動式 Session 1
ESP32 serial：COM5
Remote staging：C:\esp32_flash
```

工作排程應：

- 以互動登入使用者執行，而不是 LocalSystem/Windows Service。
- 使用者登入時自動啟動。
- 不限制最長執行時間。
- 傳入 runtime `--phone` 與 `--token`。
- 與 SSH session 脫離。

Agent 狀態至少確認：

```text
online: true
computerName: SFAA
userName: CS
agentVersion: 0.2.3
relayUrl: wss://mcp.james-huang.org/bridge/ws
```

## Screenshot and Screen Text Reading

### Screenshot

舊版以 PowerShell 擷取畫面，曾被防毒阻擋：

```text
ScriptContainedMaliciousContent
```

修正方式是 Agent 內建 C#：

- `System.Windows.Forms.Screen.PrimaryScreen.Bounds`
- `System.Drawing.Bitmap`
- `Graphics.CopyFromScreen()`
- JPEG encoding

不要退回 PowerShell 截圖腳本。

### `windows_read_screen`

功能流程：

1. 擷取目前被控 Windows 桌面。
2. 使用 Windows 內建 OCR 辨識。
3. 回傳 `fullText`。
4. 回傳每段文字的 `Text/X/Y/Width/Height`，供 UI 定位。

成功判定應包含：

```text
ok: true
width/height 正確
exitCode: 0
error: null
items 或 fullText 有合理內容
```

不能只看到 process exit code 0 就判定 OCR 成功。

舊版 PowerShell 5.1 的錯誤：

```text
DataWriterStoreOperation does not contain a method named 'AsTask'
System.__ComObject does not contain a method named 'AsTask'
```

WinRT async 等待不可直接假設 `.AsTask()` 可用。PowerShell 載入 WinRT 也可能產生無害 CLIXML progress stderr；不可把「任何 stderr」一律視為失敗，但若 stderr 含 `AsTask`、null decoder、RecognizeAsync exception，就必須失敗。

## Multiple-Agent Pitfall

手機 Relay 目前只保存一條 Agent WebSocket。GRAM 與 SFAA 同時連入時，後連線者會覆蓋前者，造成工具操作錯誤電腦。

呼叫任何截圖、讀屏、命令前先執行：

```text
windows_status
```

確認：

- `online == true`
- `computerName` 是預期目標
- `agentVersion >= 0.2.3`

已知舊端：

```text
GRAM / HCH / Agent 0.2.0
```

GRAM 0.2.0 會讓 `windows_read_screen` 出現 `.AsTask()` 錯誤。若停用 GRAM 後狀態顯示 offline 且仍殘留 GRAM metadata，需讓 SFAA Agent 重新連線或重啟其工作排程，再重查狀態。

長期修正是 Relay 按 `computerName` 保存多台 Agent，工具加入明確目標參數，而不是使用全域單一 socket。

## App Build and Deployment

```powershell
cd D:\Github\esp32_hid_bridge
flutter build apk --debug
adb -s <phone-adb-serial> install -r build\app\outputs\flutter-apk\app-debug.apk
adb -s <phone-adb-serial> shell am force-stop com.esp32bridge.esp32_hid_bridge
adb -s <phone-adb-serial> shell monkey -p com.esp32bridge.esp32_hid_bridge -c android.intent.category.LAUNCHER 1
```

手機 ADB serial/IP 可能改變；執行前用 `adb devices` 確認，不要硬猜 Wi-Fi 位址。

安裝後驗證 MCP `tools/list`：

- 不應再出現 `agent_ocr_screen`、`win_ocr_screen` 等重複名稱。
- 應只出現 `windows_read_screen`。
- 再以 `windows_status` 確認目標後實際呼叫工具。

## ESP32 HID Rules

目前英文輸入策略是 Caps Lock LED 回報模式：

1. 讀取 Windows 回傳的 Caps Lock 實際狀態。
2. 未開啟時先開啟 Caps Lock。
3. Caps Lock 開啟時讓中文輸入法直接輸入英文字母。
4. 小寫字母以 `Shift + 字母` 送出。
5. 保留約 40ms key-down/key-up 間隔。
6. 輸入完成後恢復原本 Caps Lock 狀態。

不要使用 `Ctrl+Space` 猜測 IME 狀態；它只是 toggle。先前 Alt+0nnn/NumLock 方案已被 Caps Lock LED 模式取代。

Mouse Pad 兩指上下捲動使用：

```text
SCROLL <amount>
```

### 重複字母除錯

不准再靠猜測延遲盲改。對同一次短測試同步記錄：

1. 手機 App/BLE 實際送出一次或兩次。
2. ESP32 UART `APP ->` 收到一次或兩次。
3. Windows key-down/key-up 或 Raw Input 的 scan code、VK、時間戳、repeat/injected。
4. 最終文字輸出。

判斷：

- BLE/ESP32 command 兩次：查 App listener、queue、BLE retry。
- command 一次但 HID reports 兩組：修 ESP32 report generation。
- HID event 一次但文字兩次：查 Windows filter driver、協助工具或 IME。

修改後必須在被控 Windows 實機驗證；編譯成功、燒錄成功、ESP32 回 `OK` 或 WebSocket 曾連線都不算完成。

## 中文輸入法下的 HID 部署與 Base64 命令

### 問題特徵

當目標 Windows 使用中文輸入法（包含豆包輸入法）時，直接透過 HID 輸入 PowerShell 部署命令可能會改寫或遺失 `$`、`:`、`/`、反斜線、括號、引號等字元。即使把命令改成 PowerShell `-EncodedCommand`，若 Base64 payload 的大小寫或 `+`、`/`、`=` 被 HID 改寫，仍會出現：

```text
-EncodedCommand 指定的值未正確編碼。這個值必須是 Base64 編碼。
```

### 正確處理策略

1. **不要用 `Ctrl+Space` 猜測或切換 IME**：它是 toggle，不是狀態查詢；目前英文狀態下反而可能被切成中文。
2. App 端先組合完整 PowerShell script，再用 UTF-16LE 產生 Base64：
   ```dart
   powershell -NoProfile -EncodedCommand <Base64>
   ```
3. ESP32 `ALTASCII` 不得使用 `Keyboard.press()` 處理這類大小寫敏感 payload，因為 Arduino `USBHIDKeyboard::press()` 會再次套用 ASCII layout，和 Caps Lock 策略疊加後可能破壞 Base64。
4. ESP32 應使用明確的 US keyboard raw HID keycode 對應，並用 `pressRaw()`／`releaseRaw()` 明確控制 Shift。
5. 目前 firmware 的 `ALTASCII` 建議流程：
   - 讀取並保存 HID LED 回報的 Caps Lock 狀態。
   - 暫時確保 Caps Lock 開啟。
   - 大寫字母直接送 raw 字母 keycode。
   - 小寫字母送 `Shift + raw 字母 keycode`。
   - `+`、`/`、`=` 與 PowerShell 常用符號使用 US keyboard 位置的 raw keycode。
   - 完成後恢復原 Caps Lock 狀態。

### 目前已驗證的修正位置

```text
App：D:\Github\esp32_hid_bridge\lib\main.dart
Firmware：D:\Github\esp32_s3_ble_firmware\esp32_s3_ble_firmware.ino
```

App 的部署流程目前使用 PowerShell `-EncodedCommand`；firmware 的 `typeAsciiText()` 已改為 raw keycode 輸出。這兩端必須同步，只有改 App 而保留舊 firmware 仍可能造成 Base64 損壞。

### 判斷與測試

- 看到 `Base64` decode error：先比較 PowerShell 實際收到的 payload，不要先盲改 delay。
- 看到大小寫錯誤：檢查 `Keyboard.press()` 是否被用在 `ALTASCII` 路徑，以及 `Keyboard.begin()` 的 layout/Shift 疊加。
- 看到 `+`、`/`、`=` 錯誤：檢查 raw US keycode 與 Shift 映射。
- 每次 firmware 修改後先用 Arduino CLI 編譯，再確認實際 ESP32 download port；正常執行時可能只顯示 HID，必須按 `BOOT` + `RESET/EN` 進入下載模式後才會出現 USB Serial/JTAG COM port。
- 先以安全的短字串測試，例如 Base64 對應的固定 harmless PowerShell 輸出；不要一開始在高權限或有資料風險的視窗測試。

## ESP32 Arduino 建置與燒錄

### 工具與環境

- Arduino CLI：`D:\BIN\arduino-cli.exe`
- ESP32 core：`esp32:esp32`，目前已驗證版本 `3.3.11`
- 目前韌體 FQBN：`esp32:esp32:esp32s3`
- 目前已驗證可用的下載模式裝置識別：`USB JTAG/serial debug unit`，常見實際 port 需每次重新偵測，不可固定假設 `COM7`。

### 建置

使用專案外暫存目錄保存 build output：

```powershell
$task = "$env:TEMP\pi-work\esp32-ascii-hid-build"
arduino-cli compile --fqbn esp32:esp32:esp32s3 --build-path $task D:\Github\esp32_s3_ble_firmware
```

成功時應確認 compile exit code 為 `0`，並記錄程式與 RAM 使用量；完成後清理暫存目錄，除非需要交付 binary。

### 下載模式與燒錄

1. 先執行 `arduino-cli board list` 與 Windows serial/PnP 檢查。
2. 若只看到 `USB HID Keyboard`／`HID-compliant mouse`，代表 firmware 正常執行但尚未進入 download mode。
3. 按住 `BOOT`，短按 `RESET`／`EN`，再放開 `BOOT`；重新偵測實際 COM port。
4. 確認裝置是 ESP32-S3 USB Serial/JTAG 後才執行：
   ```powershell
   arduino-cli upload -p COMx --fqbn esp32:esp32:esp32s3 --input-dir $task
   ```
5. 燒錄後每個映像都應看到 `Hash of data verified`，並確認自動 reset 完成。
6. 重新執行 `arduino-cli board list`，確認 ESP32 恢復成預期 USB HID 裝置。

### 風險與限制

- 燒錄會暫時中斷 BLE 與 USB HID；不可對未確認的 COM port 執行 upload。
- 不要使用 `erase-all`，除非使用者明確要求清除整顆 Flash。
- `N16R8` 只代表容量資訊，不足以推斷完整開發板或 VBUS GPIO。
- `type`／`ALTASCII` 目前是 US-layout ASCII，不代表支援中文或任意 Unicode。

## Verification Checklist

1. `flutter analyze` 或至少 `flutter build apk --debug` 成功。
2. APK ADB 安裝成功且 package 正確。
3. MCP tools list 無重複 Windows 工具名稱。
4. `windows_status` 指向正確電腦及 Agent 版本。
5. `windows_screenshot` 回傳可解碼且尺寸正確的 JPEG。
6. `windows_read_screen` 回傳 `ok: true`、合理文字和座標。
7. Agent 跨過多次 heartbeat 仍在線。
8. 結束 SSH session 後 Agent 仍在線。
9. 手機重啟、網路切換、Windows 使用者重新登入後，Cloudflare 與 Agent 能自動恢復。
10. 涉及 HID 修正時，以 Windows 實際事件與最終輸出驗證。

## Pitfalls

- 不要把 Agent online 當成截圖、OCR 或 HID 已修好。
- 不要只依賴編譯/燒錄結果。
- 不要將手機當前 Wi-Fi IP 當固定架構。
- 不要從手機 5590 分發大型公開 EXE；使用 GitHub Release。
- 不要把 Agent 當 LocalSystem service 來做桌面截圖/OCR。
- 不要忽略 `windows_status` 的 `computerName`；單一 Relay socket 可能被別台 Agent 搶走。
- 不要重新加入功能相同的 aliases。
- 不要在公開檔案或 skill 寫入 runtime token、SSH 密碼等秘密。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
