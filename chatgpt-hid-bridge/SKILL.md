---
name: chatgpt-hid-bridge
description: 維護 Redmi 6A USB HID gadget 與 Flutter App D:\Github\chatgpt_hid_bridge，讓手機對 Windows 枚舉成鍵盤+滑鼠且同時保留 ADB；本機 HTTP :18765 給同一支手機上的 agent 用 localhost 呼叫。當使用者提到 ChatGPT HID Bridge、Redmi 6A 當鍵盤滑鼠、USB gadget hidg、HID+ADB 複合、滑鼠板、或電腦不裝軟體用手機控制 PC 時使用。
---

# ChatGPT HID Bridge（Redmi 6A USB HID）

## When to Use

使用於：

- 把 Redmi 6A 當成 Windows 的 USB 鍵盤／滑鼠，電腦不必裝軟體
- 修改或建置 `D:\Github\chatgpt_hid_bridge`
- HID 與 ADB 要同時存在、不要切 USB 模式
- 本機 ChatGPT／agent 要打 HID HTTP API
- `/dev/hidg*` 寫不進去、滑鼠板沒反應、Windows 看得到裝置但沒按鍵
- 與 Magisk root、ConfigFS `usb_gadget`、MIUI USB 用途視窗有關的此專案問題

不要用於：

- Samsung 藍牙 HID／T25 → `phone-bluetooth-hid-t25`
- `D:\Github\esp32_hid_bridge`、WindowsBridgeAgent → `esp32-hid-bridge-windows-agent`
- 只做小米 Bootloader 解鎖 → `xiaomi-mi-unlock`

## Known Device

| 項目 | 值 |
|------|-----|
| 機型 | Xiaomi Redmi 6A |
| 代號 | `cactus` |
| ADB 序號 | `084681187d2b` |
| Android / MIUI | 9 / `V11.0.8.0.PCBMIXM` |
| USB VID | `2717` |
| HID+ADB PID | `0x210C`（Caps Lock LED descriptor 版；舊版為 `0x210A`） |
| UDC | `musb-hdrc` |
| Gadget | `/config/usb_gadget/g1` |
| Magisk | 30.7，boot.img patch（recovery-patch 無效，Ramdisk 否） |
| 套件 | `com.james.chatgpt_hid_bridge` |
| HTTP | `0.0.0.0:18765` |

其他機種必須先重測 kernel ConfigFS 與 `hidg_alloc`，不可直接套用 PID／路徑。

## Project Paths

```text
D:\Github\chatgpt_hid_bridge
D:\Github\chatgpt_hid_bridge\lib\main.dart
D:\Github\chatgpt_hid_bridge\lib\deepseek_client.dart
D:\Github\chatgpt_hid_bridge\lib\settings_store.dart
D:\Github\chatgpt_hid_bridge\android\app\src\main\kotlin\com\james\chatgpt_hid_bridge\UsbHidGadget.kt
D:\Github\chatgpt_hid_bridge\android\app\src\main\kotlin\com\james\chatgpt_hid_bridge\HidApiServer.kt
D:\Github\chatgpt_hid_bridge\android\app\src\main\kotlin\com\james\chatgpt_hid_bridge\MainActivity.kt
D:\flutter                          Flutter SDK
D:\jdk                              JAVA_HOME（Temurin 17）
D:\Android                          Android SDK
C:\Users\HCH\Downloads\Redmi6A-V11.0.8.0
C:\Users\HCH\Downloads\Magisk-v30.7\Magisk-v30.7.apk
C:\Users\HCH\Downloads\MiUnlock-7.6.727.43\fastboot.exe
```

手機 busybox：`/data/adb/magisk/busybox`

## Architecture

1. **Kernel**：必須 `CONFIG_USB_CONFIGFS_F_HID`。原廠 MTK `hidg_alloc` 會忽略 ConfigFS `report_desc`，Windows 只會看到假 HID。已 patch boot.img（`hidg_alloc` 讀 opts：subclass／protocol／report_length／desc）。
2. **Gadget**：單一 `hid.gs0`，combo descriptor（Report ID 1 = 鍵盤、ID 2 = 滑鼠），`report_length=9`。鍵盤 collection 含 LED Output report，Windows 可回傳 Caps Lock LED。`ln ffs.adb` → **HID + ADB 複合**，PID `0x210C`。
3. **App**：開啟即自動 `UsbHidGadget.start()`，無啟動／停止按鈕。Watchdog 維持 `hid.gs0` + `ffs.adb`。停用 MIUI `UsbModeChooserActivity`／Receiver，避免選 MTP 拆掉 HID。
4. **寫入**：只寫 **字元裝置** `/dev/hidgN`（常見是 `/dev/hidg2`，不是 hidg0）。`printf > /dev/hidg0` 若節點不存在會建成普通檔，滑鼠完全沒反應。
5. **滑鼠**：Dart 倍率 **1.8**；native buffer 累積位移，每 8ms 送一包 ±127，剩餘留在 buffer。滑鼠板雙擊 = 左鍵。
6. **英文輸入／中文 IME**：讀 HID Output 的 Caps Lock LED；打 ASCII 前暫時確認 Caps Lock 開啟，Caps Lock 狀態下小寫用 `Shift + raw letter`、大寫直接 raw letter，完成後恢復原本 Caps Lock。**不可點 Shift 或 Ctrl+Space 猜中英狀態**，它們都是 toggle。
7. **AI**：Flutter 直接呼叫 OpenAI-compatible DeepSeek API；設定頁管理 API Base URL、API key、模型清單（`GET /models`）。AI 分頁保留最近多輪對話並輸出 `reply` + 可選 HID `actions`。不要把 key 寫進 source、skill、log 或版本控制。

## App 內 DeepSeek AI

- AI 分頁是**多輪一問一答**，不是一次性命令；可「清除對話」。請保留最近約 24 則 user/assistant history，避免 request 無限制成長。
- 模型應由目前 Base URL 的 `GET /models` 取得，結果作下拉選單；API endpoint 可能是 `<base>/models` 或 `<base>/v1/models`。
- AI system prompt 強制 JSON：`{"reply":"中文回覆","actions":[...]}`。純聊天回空 `actions`；需要控制電腦才輸出 HID 動作。
- 語音按鈕使用 Android `RecognizerIntent`、`RECORD_AUDIO`，語言 `zh-TW`，結果填回 AI 對話輸入框。
- API key 只能存於 App sandbox 的 `SharedPreferences`；畫面須遮蔽，不可硬編碼在新版本 source。

## ChatGPT 怎麼連

官方 ChatGPT App／Custom GPT Actions 在**雲端**跑，`localhost` 不是這支手機。

| 呼叫端 | URL |
|--------|-----|
| 同一支手機上的 agent／Termux／本機 HTTP | `http://127.0.0.1:18765` |
| 同一 Wi-Fi 的電腦／雲端 Actions | `http://手機IP:18765` |

```bash
curl -s http://127.0.0.1:18765/status
curl -s -X POST http://127.0.0.1:18765/hid/type -H "Content-Type: application/json" -d "{\"text\":\"hello\"}"
curl -s -X POST http://127.0.0.1:18765/hid/key  -H "Content-Type: application/json" -d "{\"name\":\"win+r\"}"
curl -s -X POST http://127.0.0.1:18765/hid/move -H "Content-Type: application/json" -d "{\"dx\":40,\"dy\":0}"
curl -s -X POST http://127.0.0.1:18765/hid/click -H "Content-Type: application/json" -d "{\"button\":1}"
```

OpenAPI：`/openapi.json`。官方 App 不能當 HID 控制器，必須另做 bridge（本 App）。

## Inputs and Outputs

### Inputs

- Redmi 6A、ADB 序號 `084681187d2b`、目前 APK 與 HID gadget 狀態。
- Flutter／Android 原始碼、裝置端 `/dev/hidg*`、ConfigFS 與 Windows 裝置管理員的實際結果。

### Outputs

- 建置或修復後的 HID + ADB App／設定。
- 鍵盤、滑鼠、Caps Lock LED、中文輸入與 HTTP API 的實際驗證結果。
- 若無法修復，清楚指出是 App、kernel、ConfigFS、USB、ADB 或 Windows HID cache 哪一層失敗。

## Procedure

### 建置／安裝

```bat
set JAVA_HOME=D:\jdk
set ANDROID_HOME=D:\Android
set PATH=D:\flutter\bin;D:\jdk\bin;%PATH%
cd D:\Github\chatgpt_hid_bridge
flutter build apk --debug
adb -s 084681187d2b install -r build\app\outputs\flutter-apk\app-debug.apk
adb -s 084681187d2b shell am start -n com.james.chatgpt_hid_bridge/.MainActivity
```

開 App 後標題應變成 **HID + ADB**（自動啟動，約數秒）。不要點 MIUI「USB 的用途」裡的傳輸檔案。

### 驗證複合裝置

Windows 應同時有：

```text
HID Keyboard Device    HID\VID_2717&PID_210C ... COL01  Usage 0x06
HID-compliant mouse    HID\VID_2717&PID_210C ... COL02  Usage 0x02
ADB Interface          USB\VID_2717&PID_210C&MI_01
adb devices            084681187d2b device
```

手機：

```bash
adb -s 084681187d2b shell "su -c 'ls -l /dev/hidg*; ls -l /config/usb_gadget/g1/configs/b.1/'"
```

必須看到 **字元裝置** `crw`（例如 `hidg2`），且：

```text
f1 -> .../hid.gs0
f2 -> .../ffs.adb
```

### 測滑鼠／鍵盤（root，對字元節點）

```bash
# 滑鼠：Report ID 2，dx=0x50
D=$(su -c 'for n in /dev/hidg*; do [ -c "$n" ] && echo $n && break; done')
su -c "/data/adb/magisk/busybox printf '\\x02\\x00\\x50\\x00\\x00\\x00\\x00\\x00\\x00' > $D"
```

游標應移動。鍵盤 boot 8-byte 測試（舊腳本、無 Report ID）只適用單鍵盤 gadget，現行 combo 必須帶 Report ID。

## Rules and Limitations

- 只對已授權的 Redmi 6A 與 `D:\Github\chatgpt_hid_bridge` 進行修改；其他機型必須先重新驗證 kernel、ConfigFS 與 HID descriptor。
- 不得把 API key 寫入 source、Skill、log 或版本控制；測試輸出也要遮蔽 credential。
- 不得在未備份 boot image、App 設定與目前 HID 狀態前刷寫 kernel 或改動 USB gadget。
- 不得把普通檔案誤當成 `/dev/hidgN` 字元裝置，也不得用會拆掉 ADB 的 USB 設定取代現行複合 gadget。
- 未完成 Windows 鍵盤、滑鼠、ADB、中文 IME 與 HTTP API 驗證前，不得宣稱 HID + ADB 已修復。

## Pitfalls

- **MIUI USB 用途**：選 MTP 會變成 `mtp,adb`，HID 消失。App 會 `pm disable` chooser；不要再手動選傳檔。
- **不要 `setprop sys.usb.config=hid`**：Xiaomi 這條會把 protocol/subclass 打成 0。
- **不要 `setprop sys.usb.config none` 當常態**：會拆 ADB。複合綁定只動 ConfigFS UDC。
- **`/dev/hidg0` 常不是真節點**。先 `[ -c /dev/hidg* ]`。普通檔 `rw-rw-rw-` size 4 是誤寫產物，刪掉。
- App 程序不能在 Enforcing 下寫 hidg；啟動時 `setenforce 0` + `chmod 666` 後用 `FileOutputStream`，失敗再 `su printf`。
- Magisk 允許 uid `10149`（此 App）。第一次仍可能跳超級使用者 toast。
- DEBUG banner 會擋住右上角；啟動／停止 UI 已移除。
- Combo 寫入長度固定 **9 bytes**：鍵盤 `[1, mod, 0, key, 0,0,0,0,0]`，滑鼠 `[2, btn, dx, dy, wheel, 0,0,0,0]`。
- **中文輸入法不能靠 Shift／Ctrl+Space 切換**：會因未知目前狀態反向切錯（如 `cmd` 被注音轉成中文）。使用 descriptor LED + Caps Lock 回報；若 host 沒回 LED，須回報失敗，不可假裝已切英文。
- 更新 LED descriptor 後 PID 必須換新（目前 `210C`）並 rebind，舊 Windows HID cache 才會丟掉。
- 官方 ChatGPT 不能打 localhost；同一支手機的 **本機 agent 才可以**。

## Kernel patch（已完成，勿重複亂刷）

- `hidg_alloc` 原忽略 ConfigFS，descriptor 變成 7-byte stub。
- 已 patch 並 `fastboot flash boot`；Magisk root 仍在。
- 相關檔：`C:\Users\HCH\Downloads\Redmi6A-V11.0.8.0\kscan\`、`newboot.img`。
- 不要為了 HID 刷隨機 TWRP。

## Verification

1. 開 App → 標題 `HID + ADB`。
2. `adb devices` 仍有 `084681187d2b`。
3. Windows 裝置管理員：鍵盤 + 滑鼠 + ADB，VID_2717 PID_210C。
4. 滑鼠板拖曳，游標移動（速度約 1.8×）；滑鼠板雙擊會送左鍵。
5. 用 Windows 中文 IME 實測輸入 `cmd`、`notepad`；不得變成注音候選字。完成後 Caps Lock 要回到原狀。
6. 本機：`curl http://127.0.0.1:18765/status`（在手機上）回 `enabled: true`；確認 `caps` 值會隨 Windows Caps Lock LED 變動。
7. AI 設定頁成功取得模型列表；AI 頁能保留前一輪上下文，語音辨識結果能填入對話輸入框。
