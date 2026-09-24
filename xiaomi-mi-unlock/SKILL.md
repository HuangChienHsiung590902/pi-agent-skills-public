---
name: xiaomi-mi-unlock
description: 管理與排查 Xiaomi/Redmi 手機的 Bootloader 解鎖流程：透過 ADB 檢查裝置、OEM 解鎖允許與實際鎖定狀態，從小米官方來源下載/更新 Mi Unlock，處理舊版登入白畫面，並準備 Fastboot 解鎖。當使用者提到小米、Redmi、Mi Unlock、Bootloader、OEM 解鎖、解鎖工具不能登入或要刷機前檢查時使用。
---

# Xiaomi / Redmi Mi Unlock

## When to Use

使用於：

- 檢查 Xiaomi/Redmi 手機是否已連上 ADB。
- 分辨「已開啟 OEM 解鎖允許」與「Bootloader 已真正解鎖」。
- 下載或更新官方 Mi Unlock 工具。
- 舊版 Mi Unlock 登入頁面白畫面、登入元件失效。
- 準備進入 Fastboot 並使用 Mi Unlock 解鎖。

本技能目前的實測裝置是 Xiaomi Redmi 6A，裝置代號 `cactus`，ADB 序號 `084681187d2b`；其他 Xiaomi 裝置必須先重新偵測序號與屬性，不可直接套用這些值。

## Safety Boundaries

- **不要自動按下 Mi Unlock 的 Unlock。** Bootloader 解鎖通常會清除手機全部資料，且是不可逆或高風險操作；必須由使用者確認已備份並明確授權後才可繼續。
- 不要嘗試繞過 Xiaomi 帳號、解鎖等待時間、伺服器權限或帳號驗證。
- 不要把 Xiaomi 帳號、密碼、驗證碼、Cookie 或 token 寫入 skill、日誌或回覆。
- 只從 Xiaomi/MIUI 官方網域取得 Mi Unlock；第三方改包不可當成官方工具。

## Known Device Evidence

曾實測的 Redmi 6A 狀態：

- Android 9
- MIUI `V11.0.8.0.PCBMIXM`
- 安全性更新：2020-05-01
- `ro.boot.flash.locked=1`
- `ro.boot.vbmeta.device_state=locked`
- `ro.boot.verifiedbootstate=green`
- `sys.oem_unlock_allowed=1`（後來已開啟 OEM 解鎖允許）

上述表示：OEM 解鎖設定已允許，但 Bootloader 仍然鎖定。每次操作都要以目前的 ADB 輸出為準。

## Procedure

### 1. 找到目前的 ADB 裝置

```bash
adb devices -l
```

裝置必須顯示為 `device`。若是 `unauthorized`，解鎖並在手機上接受 USB 偵錯 RSA 指紋；若清單為空，先處理 USB 線、驅動程式與 USB 偵錯。

取得裝置資訊：

```bash
adb -s <serial> shell 'printf "manufacturer="; getprop ro.product.manufacturer; printf "model="; getprop ro.product.model; printf "device="; getprop ro.product.device; printf "android="; getprop ro.build.version.release; printf "miui="; getprop ro.miui.ui.version.name; printf "build="; getprop ro.build.display.id'
```

### 2. 分開檢查設定與實際 Bootloader 狀態

```bash
adb -s <serial> shell 'printf "development_settings_enabled="; settings get global development_settings_enabled; printf "adb_enabled="; settings get global adb_enabled; printf "oem_unlock_enabled="; settings get global oem_unlock_enabled; printf "sys.oem_unlock_allowed="; getprop sys.oem_unlock_allowed; printf "flash_locked="; getprop ro.boot.flash.locked; printf "vbmeta_state="; getprop ro.boot.vbmeta.device_state; printf "verified_boot="; getprop ro.boot.verifiedbootstate'
```

判讀：

- `development_settings_enabled=1`：開發者選項已啟用。
- `adb_enabled=1`：USB 偵錯已啟用。
- `sys.oem_unlock_allowed=1`：目前允許 OEM 解鎖；這不等於 Bootloader 已解鎖。
- `ro.boot.flash.locked=0` 或 `ro.boot.vbmeta.device_state=unlocked`：通常表示已解鎖。
- `ro.boot.flash.locked=1`、`ro.boot.vbmeta.device_state=locked`：仍然鎖定。
- `ro.boot.verifiedbootstate=green`：目前是驗證啟動狀態，常見於仍鎖定的原廠系統。

`settings get global oem_unlock_enabled` 在部分 MIUI 版本可能回傳 `null`，不能只靠這一項判斷；以 `sys.oem_unlock_allowed` 與 Bootloader 屬性合併判讀。

### 3. 使用官方更新頁取得 Mi Unlock

官方英文下載頁：

```text
https://en.miui.com/unlock/download_en.html
```

官方中文頁的目前下載連結曾指向：

```text
https://cdn.cnbj1.fds.api.mi-img.com/flash-tool/miflash_unlock_7.6.727.43.zip
```

下載後保存到：

```text
C:\Users\HCH\Downloads\miflash_unlock_7.6.727.43.zip
```

在 Windows Git Bash 可使用：

```bash
mkdir -p "$HOME/Downloads"
curl -L --fail --retry 2 \
  -o "$HOME/Downloads/miflash_unlock_7.6.727.43.zip" \
  "https://cdn.cnbj1.fds.api.mi-img.com/flash-tool/miflash_unlock_7.6.727.43.zip"
```

解壓縮：

```bash
mkdir -p "$HOME/Downloads/MiUnlock-7.6.727.43"
unzip -oq "$HOME/Downloads/miflash_unlock_7.6.727.43.zip" \
  -d "$HOME/Downloads/MiUnlock-7.6.727.43"
```

執行檔通常是：

```text
C:\Users\HCH\Downloads\MiUnlock-7.6.727.43\miflash_unlock.exe
```

### 4. 驗證下載檔

至少確認 ZIP 格式與壓縮檔測試：

```bash
file "$HOME/Downloads/miflash_unlock_7.6.727.43.zip"
unzip -t "$HOME/Downloads/miflash_unlock_7.6.727.43.zip"
```

本次實測檔案 SHA-256 為：

```text
99a3cd6186a9135c319652556cfb8c176743844af1b63070b81f9f4919d064b0
```

雖然檔案已通過 ZIP 測試，仍應以官方頁面目前提供的版本為準；若官方連結或雜湊改變，不要硬套用這個歷史雜湊。

### 5. 舊版登入白畫面的處理

舊版 `Mi Unlock 6.5.224.28` 曾出現：

- Disclaimer 頁面可顯示。
- 進入 `Account Authentication` 後整個登入區域白畫面。
- 舊版畫面顯示可更新到 `7.6.727.43`。

這通常是舊版內嵌登入元件或舊服務端流程失效，不要修改帳號資料或嘗試繞過登入。關閉舊版，改用官方新版 `7.6.727.43`；若新版也不能登入，檢查 Windows 時間、網路、代理/防火牆、瀏覽器登入與 Xiaomi 帳號本身，並以官方解鎖頁 FAQ 為準。

### 6. 準備 Fastboot（僅在使用者已備份且明確要操作時）

一般官方流程：

1. 在手機確認 OEM 解鎖允許、USB 偵錯、Mi 帳號與裝置綁定狀態。
2. 關機。
3. 按住音量下鍵與電源鍵進入 Fastboot。
4. 用 USB 連接電腦。
5. 開啟 Mi Unlock，登入同一個 Xiaomi 帳號。
6. 確認工具顯示正確裝置後，再由使用者自行判斷是否按 Unlock。

進入 Fastboot 後可先唯讀檢查，不進行解鎖：

```bash
fastboot devices
fastboot getvar product 2>&1
fastboot getvar unlocked 2>&1
```

若 `fastboot devices` 沒有裝置，先安裝或修復 Mi Unlock 資料夾內的 `driver_install_64.exe`，再重新插拔 USB；不要在裝置未正確辨識時嘗試解鎖。

## Pitfalls

- **OEM 解鎖允許 ≠ Bootloader 已解鎖。** 必須分別檢查 `sys.oem_unlock_allowed` 與 `ro.boot.*`。
- `adb devices` 顯示 `device` 只代表 ADB 授權成功，不代表 Bootloader 狀態。
- 舊版 Mi Unlock 的登入白畫面不代表帳號錯誤；先更新到官方目前版本。
- 不要使用來路不明的「免等待」「繞過帳號」解鎖工具。
- Bootloader 解鎖通常會 factory reset；在沒有備份、未確認照片/聯絡人/訊息同步前，不要按 Unlock。
- Mi Unlock 可能要求等待時間或顯示帳號與手機綁定不符；只能依官方流程等待或重新確認綁定，不能用腳本偽造結果。
- 直接用 ADB 修改受保護的 Bootloader 狀態不可靠，也不能取代官方 Mi Unlock。
- 不要將登入密碼或驗證碼輸入到 shell 命令、skill 檔案或截圖紀錄。

## Verification

完成下載/準備後確認：

1. `adb devices -l` 顯示目標手機為 `device`。
2. 已記錄當前 `model`、`device`、Android/MIUI 版本。
3. 已取得 `sys.oem_unlock_allowed` 與 `ro.boot.flash.locked`/`ro.boot.vbmeta.device_state` 的目前值。
4. Mi Unlock ZIP 通過 `unzip -t`，且解壓後存在 `miflash_unlock.exe`。
5. 若只是準備解鎖，確認沒有執行 Unlock、沒有重置手機、沒有寫入分割區。
6. 真正解鎖後，重新開機並用 ADB/Fastboot 重新讀取 `ro.boot.vbmeta.device_state`；只有看到 `unlocked` 等明確證據才宣稱成功。

## Conformance Addendum

## Inputs and Outputs
- **Input:** Xiaomi/Redmi 裝置的 ADB/Fastboot 識別資訊、目前系統屬性、官方 Mi Unlock 下載來源，以及使用者對備份與解鎖的明確授權。
- **Output:** 可驗證的裝置狀態、官方工具下載/解壓結果、登入或 Fastboot 準備狀態；不在未授權下執行破壞性解鎖。

## Rules and Limitations
- 路徑、版本、下載 URL 與裝置序號可能變動；以目前官方頁面與目前裝置輸出為準。
- 不公開或保存 Xiaomi 帳號憑證。
- 不繞過 Xiaomi 的安全機制、帳號綁定或等待限制。
- 不把「下載工具」或「進入 Fastboot」描述成「已完成解鎖」。
