---
name: esp32-ble-wifi-bridge-mvp
description: >
  建置、燒錄、安裝與除錯 D:\Github\esp32_ble_wifi_bridge_mvp：手機 Flutter App 先用 BLE provision Wi-Fi SSID/password 給 ESP32-S3，再切到 Wi-Fi/HTTP 控制 /status 與 /gpio。當使用者提到 ESP32-IO-BRIDGE、esp32_ble_wifi_bridge_mvp、BLE 配網、Flutter 控制 ESP32 GPIO、ESP-IDF build/flash、App 找不到 ESP32、COM7 CH343、MSYSTEM ESP-IDF 錯誤，或要重跑這套手機控制 ESP32 MVP 時載入。
---

# ESP32 BLE Wi-Fi Bridge MVP

## When to Use

使用於這套特定 MVP 的開發、建置、燒錄、App 安裝與實機診斷：

- 專案路徑：`D:\Github\esp32_ble_wifi_bridge_mvp`
- Flutter App：`D:\Github\esp32_ble_wifi_bridge_mvp\flutter_app`
- ESP-IDF 韌體：`D:\Github\esp32_ble_wifi_bridge_mvp\esp32_idf`
- Arduino 參考韌體：`D:\Github\esp32_ble_wifi_bridge_mvp\esp32_arduino\esp32_ble_wifi_bridge_mvp.ino`
- BLE advertised name：`ESP32-IO-BRIDGE`
- 手機作為「腦」，ESP32 作為外部硬體 I/O 擴充。
- 流程：Flutter App → BLE 寫 Wi-Fi credentials → ESP32 連 Wi-Fi/手機熱點 → App 用 Wi-Fi HTTP 控制 GPIO/status。

不要用於：

- `D:\Github\esp32_hid_bridge` 的 HID/WindowsBridgeAgent/Cloudflare relay 維護；改用 `esp32-hid-bridge-windows-agent` 或相關 HID skill。
- `D:\Github\wolfssh_echoserver` 的 wolfSSH、USB Host、BLE HID 韌體；改用 `esp32-s3-wolfssh-usb-ble-hid`。

## Inputs and Outputs

### Inputs

- 使用者要做的動作：建置、燒錄、安裝 APK、配網、查 status、GPIO 測試或除錯。
- ESP32 實際序列埠清單，尤其要分辨 `COM7 - USB-Enhanced-SERIAL CH343` 與 `COM5 - SAMSUNG Mobile USB Modem`。
- Android ADB 裝置狀態。
- Flutter App 畫面 log、Android uiautomator dump、ADB logcat、ESP-IDF monitor log。
- Wi-Fi SSID/password 只可在當次操作使用，不要寫入 Skill 或長期記憶。

### Outputs

- 已 build 的 ESP-IDF 韌體與大小檢查。
- 經確認後燒錄到正確 ESP32 COM port 的結果。
- 已 build 並透過 ADB 安裝的 debug APK。
- BLE 配網、Wi-Fi 連線、HTTP `/status` 與 `/gpio` 控制的驗證證據。
- 若失敗，提供分層診斷：App 掃描、BLE advertising、BLE GATT、Wi-Fi STA、HTTP API、GPIO allowlist。

## Authoritative Paths

```text
Project root: D:\Github\esp32_ble_wifi_bridge_mvp
ESP-IDF project: D:\Github\esp32_ble_wifi_bridge_mvp\esp32_idf
Flutter app: D:\Github\esp32_ble_wifi_bridge_mvp\flutter_app
ESP-IDF install: D:\Espressif\v5.5.2\esp-idf
Skill scripts: scripts/
Protocol reference: references/protocol.md
Troubleshooting reference: references/troubleshooting.md
```

## Procedure

### 1. 先確認現況與避免誤改舊專案

1. 讀取/檢查任務相關檔案，不憑摘要猜程式碼。
2. 執行：
   ```powershell
   git -C D:\Github\esp32_ble_wifi_bridge_mvp status --short
   ```
3. 不要修改 `D:\Github\esp32_hid_bridge` 或 `D:\Github\wolfssh_echoserver`，除非使用者明確要求。
4. 若要刷機，先查 Windows serial ports；不要把 Samsung modem 當 ESP32。

### 2. 建置 ESP-IDF 韌體

優先用本 Skill 腳本：

```powershell
pwsh -File D:\OB\skills\esp32-ble-wifi-bridge-mvp\scripts\build-idf.ps1
```

手動流程：

```powershell
Remove-Item Env:MSYSTEM -ErrorAction SilentlyContinue
cd D:\Github\esp32_ble_wifi_bridge_mvp\esp32_idf
& D:\Espressif\v5.5.2\esp-idf\export.ps1
idf.py set-target esp32s3
idf.py build
```

必要設定與已知修正：

- `main\CMakeLists.txt` 需要 `esp_driver_gpio`。
- `sdkconfig.defaults` 使用 BLE 4.2 legacy advertising，關閉 BLE 5.0 extended advertising。
- 使用 `partitions.csv`，factory app partition 大小 `0x300000`，避免 BLE+Wi-Fi+HTTP binary 超過預設 1MB。
- `CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192`，不要用過期的 `CONFIG_MAIN_TASK_STACK_SIZE`。

### 3. 確認 ESP32 COM port 並燒錄

先列出連接埠：

```powershell
Get-CimInstance Win32_SerialPort | Select-Object DeviceID,Name,Description
Get-PnpDevice -Class Ports -ErrorAction SilentlyContinue | Select-Object Status,FriendlyName,InstanceId
```

常見本機結果：

```text
COM7 - USB-Enhanced-SERIAL CH343 -> ESP32-S3 UART，通常可用於 flash
COM5 - SAMSUNG Mobile USB Modem -> 手機，不可當 ESP32 燒錄
COM6 - USB Serial Device VID_303A PID_1001 -> 可能是 ESP32-S3 USB/JTAG/serial，需依實測確認
```

經確認後用腳本燒錄：

```powershell
pwsh -File D:\OB\skills\esp32-ble-wifi-bridge-mvp\scripts\flash-idf.ps1 -Port COM7
```

手動燒錄：

```powershell
Remove-Item Env:MSYSTEM -ErrorAction SilentlyContinue
cd D:\Github\esp32_ble_wifi_bridge_mvp\esp32_idf
& D:\Espressif\v5.5.2\esp-idf\export.ps1
idf.py -p COM7 flash
```

若要開 monitor：

```powershell
pwsh -File D:\OB\skills\esp32-ble-wifi-bridge-mvp\scripts\monitor-idf.ps1 -Port COM7
```

成功 boot 關鍵 log：

```text
ESP-IDF BLE Wi-Fi Bridge MVP booting
wifi_manager: {"state":"wifi_ready"}
BLE advertising as ESP32-IO-BRIDGE
ble_prov: status: {"state":"ble_ready"}
```

### 4. Build 並安裝 Flutter Android App

優先用腳本：

```powershell
pwsh -File D:\OB\skills\esp32-ble-wifi-bridge-mvp\scripts\build-install-app.ps1
```

手動流程：

```powershell
cd D:\Github\esp32_ble_wifi_bridge_mvp\flutter_app
flutter analyze
flutter build apk --debug
adb install -r build\app\outputs\flutter-apk\app-debug.apk
```

成功輸出：

```text
No issues found!
✓ Built build\app\outputs\flutter-apk\app-debug.apk
Performing Streamed Install
Success
```

### 5. 實機測試流程

1. 手機打開 `ESP32 BLE WiFi Bridge`。
2. 點「掃描並連線 ESP32」。
3. 應找到 BLE MAC 類似 `7C:4F:AD:B6:21:86`，並顯示 `BLE 已連線，可送 Wi-Fi 設定`。
4. 輸入 Wi-Fi SSID/password，點「透過 BLE 寫入 Wi-Fi 設定」。
5. 等 ESP32 BLE notify：
   ```json
   {"state":"connected","ip":"10.x.x.x","port":80}
   ```
6. App 用該 IP 呼叫：
   ```http
   GET http://<esp-ip>/status
   GET http://<esp-ip>/gpio?pin=2&value=1
   GET http://<esp-ip>/gpio?pin=2&value=0
   ```
7. 成功例：
   ```json
   {"device":"ESP32-IO-BRIDGE-IDF","wifi":"connected","ssid":"XXX","ip":"10.28.130.211","uptime_ms":366176}
   {"ok":true,"pin":2,"value":1}
   ```

### 6. 從手機畫面確認狀態

若使用者問「你看一下手機」或要確認 App 是否連上，可用：

```bash
adb shell dumpsys window | grep -E 'mCurrentFocus|mFocusedApp'
adb exec-out uiautomator dump /dev/tty 2>/dev/null
adb logcat -d -t 300 | grep -iE 'esp32|flutter|bluetooth|gatt|ESP32-IO-BRIDGE'
```

判斷已成功的畫面/文字：

```text
ESP32 IP：<ip>
BLE 狀態：{"state":"connected","ip":"<ip>","port":80}
HTTP /status 200
GPIO 2=1 200
GPIO 2=0 200
```

## Rules and Limitations

- 不要把 Wi-Fi 密碼寫入 Skill、README、commit 或長期記憶。
- GPIO 只控制 firmware allowlist 內腳位；避免 boot/flash/USB/strapping 危險腳位。
- 刷機前必須確認 port，不要直接對 `COM5 - SAMSUNG Mobile USB Modem` 操作。
- BLE provisioning 目前是 MVP，尚未有正式安全設計；不可宣稱已安全防陌生人控制。
- 正式產品化前應加入：實體按鈕進入 provisioning mode、timeout、bonding/pairing 或 ECDH+AES、device token、NVS credentials persistence、命令 allowlist。
- 若 Android App 找不到 ESP32，先不要重刷；先查 BLE advertising 與 App 掃描 filter。

## Pitfalls

- ESP-IDF 在 Git Bash/MSYS 環境會因 `MSYSTEM` 失敗：必須移除 `Env:MSYSTEM` 後再跑 `install.ps1` / `export.ps1` / `idf.py`。
- `idf.py build` 若找不到 Python venv：重新跑 `D:\Espressif\v5.5.2\esp-idf\install.ps1`。
- ESP32 advertising log 若出現 `Partial data write into ADV`，Android 用 service UUID filter 可能掃不到；Flutter App 應全量掃描後再用名稱/MAC/UUID 在 Dart 端判斷。
- BLE 5.0 extended advertising 與 legacy GAP API 混用可能導致 link/config 問題；本 MVP 目前固定 BLE 4.2 legacy advertising。
- 預設 partition 太小會造成 app image size error；使用 `partitions.csv` 的 `factory` `0x300000`。
- `COM7` 是本機常見值，不是保證值；每次接板後都要重新查。
- `idf_monitor` 用 timeout 結束會回報 exit code 143/124，若 log 已看到 boot/BLE ready，這不是韌體失敗。

## Verification

完成一次維護/部署後至少驗證：

1. ESP-IDF build 成功：
   ```text
   Project build complete.
   ```
2. 若有燒錄，確認：
   ```text
   Chip is ESP32-S3
   Hash of data verified.
   Done
   ```
3. Monitor 看到：
   ```text
   BLE advertising as ESP32-IO-BRIDGE
   {"state":"ble_ready"}
   ```
4. Flutter App：
   ```text
   flutter analyze -> No issues found!
   adb install -> Success
   ```
5. 實機 App log 顯示：
   ```text
   BLE 狀態：{"state":"connected","ip":"<esp-ip>","port":80}
   HTTP /status 200
   GPIO <pin>=<value> 200
   ```
6. 若更新本 Skill，必須執行 skills 索引重建與 audit。

更多通訊細節見 `references/protocol.md`；常見錯誤處理見 `references/troubleshooting.md`。
