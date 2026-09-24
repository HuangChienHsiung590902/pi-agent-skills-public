---
name: esp32-s3-wolfssh-usb-ble-hid
description: >
  維護、建置、燒錄及除錯 D:\Github\wolfssh_echoserver 的 ESP32-S3 N16R8 韌體時使用；涵蓋 ESP-IDF 5.5.2、wolfSSH 受限遠端 Shell、Wi-Fi SoftAP、原生 USB OTG Host MSC/FAT16/FAT32 診斷，以及 BLE 複合 HID 鍵盤滑鼠。當使用者提到 wolfssh_echoserver、COM7、ESP32-S3 USB 隨身碟無法枚舉、usb_devices=0、ESP32-S3 Remote HID、hid_started/hid_connected、遠端 type/key/combo/move/click，或要求重建／燒錄這套韌體時載入。
---

# ESP32-S3 wolfSSH USB Host + BLE HID

## When to Use

使用於這套特定 ESP32-S3 韌體的開發與維護：

- 修改或重建 `D:\Github\wolfssh_echoserver`。
- 維護 wolfSSH 受限命令、USB Host MSC、FAT 唯讀檔案操作或 BLE HID。
- 經 CH343 UART 燒錄 ESP32-S3，預設常見連接埠為 `COM7`，但每次都必須重新確認。
- 診斷 `usb_devices=0`、`last_address=0`、磁碟未掛載或 Native USB OTG 無枚舉。
- 診斷 `hid_started=no`、`hid_connected=no`、BLE HID 不廣播、無法配對或鍵鼠報告未送出。
- 增加 `type`、`key`、`combo`、`move`、`click` 等明確 allowlist 命令。

不要用於 `D:\Github\esp32_hid_bridge` 的手機 App、WindowsBridgeAgent、Cloudflare relay 或 MCP；那些任務改讀 `..\esp32-hid-bridge-windows-agent\SKILL.md`。

## Inputs and Outputs

### Inputs

- 使用者要修改、建置、燒錄或診斷的具體需求。
- 目前專案檔案與 `sdkconfig`／`sdkconfig.defaults`。
- 實際 UART 連接埠、完整開發板型號、接口與接線。
- USB／BLE／wolfSSH 的即時輸出，不以過往紀錄取代實測。

### Outputs

- 經檢查後的韌體修改。
- ESP-IDF build 結果與映像大小。
- 經使用者授權後的燒錄結果。
- USB、BLE HID 與遠端 Shell 的分層診斷結論。
- 實際修改檔案、驗證證據及尚未解決的硬體／安全限制。

## Authoritative Paths

詳細檔案用途與架構見 `references/project-layout.md`。主要位置：

```text
專案：D:\Github\wolfssh_echoserver
ESP-IDF：D:\Espressif\v5.5.2\esp-idf
ESP-IDF tools：C:\Espressif\tools
本 Skill 建置腳本：scripts/build-firmware.ps1
```

修改前至少確認：

```text
main/main.c
main/usb_debug.c
main/ble_hid_control.c
main/esp_hid_gap.c
main/CMakeLists.txt
sdkconfig.defaults
partitions_16mb.csv
README_LOCAL.md
```

## Procedure

### 1. 先確認現況，不憑摘要猜程式碼

1. 讀取與任務直接相關的原始碼、CMake、partition 與設定檔。
2. 執行 `git -C D:\Github\wolfssh_echoserver status --short`，區分既有變更與本次變更。
3. 確認晶片仍是 ESP32-S3 N16R8；不要把 `N16R8` 當成完整開發板型號。
4. 需要燒錄時，先確認 Windows 裝置管理員或序列埠清單中的實際 COM port。
5. 不要改動或覆蓋另一套舊韌體 `D:\Github\esp32_s3_ble_firmware`。

### 2. 維護功能邊界

保持下列架構分離：

```text
CH343 UART / COM port -> 建置後燒錄與序列診斷
ESP32-S3 Native USB OTG GPIO19/20 -> USB Host MSC
Wi-Fi SoftAP + wolfSSH -> 遠端受限命令
BLE -> 複合 HID 鍵盤與滑鼠
```

遠端 Shell 應採明確 allowlist；磁碟預設唯讀，不加入任意 shell、刪除、格式化或寫檔。若使用者要求開放高風險能力，先說明風險並確認權限、範圍與認證方式。

### 3. USB Host MSC 診斷

依層級判斷，不要先怪 FAT32：

1. `host_started=no`：先查 USB Host driver／task 啟動。
2. `host_started=yes`、`usb_devices=0`、`last_address=0`：尚未枚舉，優先查實體層：
   - Native USB OTG 接口是否正確。
   - VBUS 對 GND 是否約 5V。
   - D- 是否接 GPIO19、D+ 是否接 GPIO20。
   - Host／OTG 轉接器及共同 GND。
   - HDD／SSD 優先使用獨立供電 USB 2.0 Hub，並避免 5V 倒灌。
3. 已有 USB address 但 MSC install 失敗：查 BOT／MSC class 與裝置相容性。
4. MSC 成功但 mount 失敗：才查 MBR、FAT16/FAT32、磁區大小與 VFS。
5. 未取得完整板型或原理圖前，不猜 VBUS enable GPIO，也不驅動未知 GPIO。

### 4. BLE HID 維護

目前 report map：

- Report ID 1：8-byte 鍵盤報告。
- Report ID 2：滑鼠按鍵、X/Y 相對位移與 wheel。

修改時維持 key-down 後必送全零 key-up；滑鼠 click 後也要釋放按鍵。`type` 目前是美式鍵盤 ASCII mapping，不能宣稱支援中文或任意 Unicode。

ESP-IDF 某些 build 未必穩定送出 `ESP_HIDD_START_EVENT`；目前實作在 `esp_hidd_dev_init()` 後延遲並明確呼叫 `esp_hid_ble_gap_adv_start()`。若 `hid_started=no`，先查初始化回傳值及廣播；若 `hid_started=yes hid_connected=no`，表示正在等 host 配對／連線。

**安全注意：**目前命令執行條件主要是 `hid_connected`。BLE connected 不等於已證明完成授權認證；在未檢查 GAP authentication/bonding event 與安全參數前，不得宣稱「只有已授權配對者才能控制」。正式部署前應補上並驗證 bonding、MITM／passkey 策略及清除 bond 流程。

### 5. 建置

建議使用 Skill 內的確定性腳本：

```powershell
pwsh -File D:\OB\skills\esp32-s3-wolfssh-usb-ble-hid\scripts\build-firmware.ps1 -Action Build
```

腳本會：

- 移除當前程序的 `MSYSTEM`，避免 Git Bash 環境干擾 ESP-IDF。
- 設定這台機器已驗證的 ESP-IDF 5.5.2 paths。
- 確認必要路徑存在。
- 執行 `idf.py build`。

若 target 或設定已污染，先人工確認，再使用：

```powershell
Remove-Item D:\Github\wolfssh_echoserver\sdkconfig -Force
Remove-Item D:\Github\wolfssh_echoserver\build -Recurse -Force
idf.py set-target esp32s3
idf.py build
```

這會刪除可重建產物與目前 `sdkconfig`；執行前必須確認使用者意圖及保留設定是否已進入 `sdkconfig.defaults`。

### 6. Partition 大小

BLE/bluedroid + wolfSSH + USB MSC 映像曾超出 `partitions_singleapp_large.csv` 的約 `0x177000` factory partition。此 N16 裝置目前使用：

```text
partitions_16mb.csv
factory size = 0x600000
```

若出現 `app partition is too small`：

1. 先看實際 binary 大小。
2. 確認 flash 真的是 16 MB。
3. 確認 `CONFIG_PARTITION_TABLE_CUSTOM=y` 與檔名一致。
4. 檢查 partitions 總範圍沒有超出 16 MB。
5. 不要只為了通過 build 就盲目放大 partition。

### 7. 燒錄

燒錄會覆寫 ESP32 flash，必須先取得使用者明確同意並確認 port：

```powershell
pwsh -File D:\OB\skills\esp32-s3-wolfssh-usb-ble-hid\scripts\build-firmware.ps1 -Action Flash -Port COM7
```

或建置後燒錄：

```powershell
pwsh -File D:\OB\skills\esp32-s3-wolfssh-usb-ble-hid\scripts\build-firmware.ps1 -Action BuildFlash -Port COM7
```

不可在未確認 COM port 時，把歷史上的 `COM7` 當成永久事實。

### 8. 實機驗證

建置成功不等於功能完成。至少依任務驗證：

1. 韌體啟動且 Wi-Fi AP 可見。
2. wolfSSH TCP port 可連線；帳密從目前專案設定或使用者安全提供，不把密碼寫入 Skill／log。
3. 遠端 `status`、`usb logs`、`disk` 結果符合實際硬體狀態。
4. `hid status` 顯示 `hid_started=yes`。
5. 在目標電腦配對後，`hid_connected=yes`。
6. 在安全測試視窗中實測 `type`、`key`、`move`、`click`；避免在登入框、管理介面或有資料損失風險的位置測試。
7. 若宣稱有 BLE 授權，必須另有 authentication/bonding event 的證據，不能只看 connected。

## Rules and Limitations

- 修改檔案前先讀取實際內容。
- 建置可直接執行；燒錄、erase、刪除 build/sdkconfig 或硬體 GPIO 試探需先確認風險與意圖。
- 磁碟功能維持唯讀，除非使用者明確要求並接受資料風險。
- 只支援 FAT16／FAT32；不可宣稱支援 exFAT／NTFS。
- `type` 只支援目前 mapping 中的 ASCII／US layout；中文 IME 與 Unicode 不在目前能力內。
- 不在 Skill、公開文件或回覆中洩漏 Wi-Fi、SSH 或其他密碼；必要時只指出應從本機目前設定安全取得。
- wolfSSH 固定測試帳密若仍存在，正式部署前必須更換；最好移入可配置的 NVS／Kconfig，而非硬編碼。
- BLE HID 與 USB Host 使用不同傳輸路徑；不要把 BLE HID 誤寫成 Native USB Device HID。
- 外部供電 Hub 可能倒灌 5V；接線前確認供電拓撲。

## Pitfalls

- Git Bash 的 `MSYSTEM` 可能干擾 ESP-IDF Windows toolchain；建置程序先移除它。
- 只看到 `USB Host installed` 不代表裝置有枚舉。
- `usb_devices=0` 時重做 FAT32 通常無效，因為尚未進入檔案系統層。
- `hid_started=yes` 只代表 HID 已啟動／廣播，不代表已連線。
- `hid_connected=yes` 不自動等於已授權／已 bonding。
- 某些 IDF build 不可靠觸發 HID START event；保留明確啟動 advertising 的處理，除非新版 API 已實測不需要。
- 修改 report ID、report map 或 payload 長度必須同步；不一致會造成 host 收到錯誤報告。
- `N16R8` 只有 flash/PSRAM 容量資訊，不足以判定開發板 VBUS switch GPIO。
- `COM7`、IP、interface 與裝置狀態都可能改變；目前證據優先於歷史值。

## Verification

完成條件依本次範圍選用，但至少包括：

1. `idf.py build` 成功。
2. `wolfssh_echoserver.bin` 未超出 custom factory partition。
3. 若有燒錄：esptool 完成 write、hash verify 與 reset。
4. 若有遠端改動：wolfSSH 實際登入並執行相關 allowlist 命令。
5. 若有 USB 改動：以 `status`／`usb logs` 證明枚舉、MSC 與 mount 所在層級。
6. 若有 HID 改動：證明 advertising、連線及實際鍵鼠輸出；必要時同時查 UART/ESP log。
7. 回報修改路徑、build/flash/test 結果與未解決限制。

## References

- 本機專案架構與已知狀態：`references/project-layout.md`
- ESP-IDF HID Device example：`D:\Espressif\v5.5.2\esp-idf\examples\bluetooth\esp_hid_device`
- ESP-IDF USB Host API：`D:\Espressif\v5.5.2\esp-idf\components\usb`
- Espressif USB Host MSC component：https://components.espressif.com/components/espressif/usb_host_msc
- wolfSSH component：https://components.espressif.com/components/wolfssl/wolfssh

本 Skill 是依本機實作整理，未直接匯入或執行外部 skills finder 候選內容。
