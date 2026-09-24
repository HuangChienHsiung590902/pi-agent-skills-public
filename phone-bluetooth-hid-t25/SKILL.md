---
name: phone-bluetooth-hid-t25
description: Use when controlling T25 from the Samsung phone via Bluetooth HID, maintaining the Flutter app D:\Github\esp32_hid_bridge that turns the phone into a Bluetooth keyboard/mouse, or reasoning about whether phone Bluetooth/USB can control T25 before Windows login or in BIOS. Covers ADB serials, app paths, tested capabilities, limitations of HID as a one-way channel, and the BIOS/USB-gadget constraints.
---

# Phone Bluetooth HID Control for T25

## What this skill is for

Use this skill when the user asks to:

- 用手機藍牙控制 T25
- 把手機當 T25 的鍵盤/滑鼠
- 操作/修改 `D:\Github\esp32_hid_bridge`
- 讓手機 HID app 開網頁、打字、移動滑鼠、按快捷鍵
- 判斷手機藍牙 HID 能不能控制 Windows 登入畫面、BIOS、開機前畫面
- 查手機接 USB 到 T25 後到底是不是 HID

Do **not** redirect the user back to ESP32 as the main plan unless explaining the difference between Bluetooth HID and USB HID/BIOS control. The user explicitly wanted to use the phone, not ESP32.

## Known devices and connection info

Phone:

- Model: Samsung Galaxy A34 5G / `SM-A346`
- USB ADB serial when attached locally: `RFCW3049BHM`
- Wi‑Fi/TCP ADB: `10.145.119.96:5555`
- ADB check:
  ```bash
  adb devices
  adb connect 10.145.119.96:5555
  ```

T25:

- IP: `10.145.119.209`
- Bluetooth device name observed from phone: `T25`
- Bluetooth MAC observed in app status: `A8:41:F4:42:28:19`
- SSH reference from other skills: port `2222`, user `Administrator`, password login; Git Bash `sshpass` is unreliable, use `plink` if needed.

## Flutter app

Project root:

```text
D:\Github\esp32_hid_bridge
```

Important files:

```text
D:\Github\esp32_hid_bridge\lib\main.dart
D:\Github\esp32_hid_bridge\android\app\src\main\kotlin\com\esp32bridge\esp32_hid_bridge\MainActivity.kt
D:\Github\esp32_hid_bridge\android\app\src\main\kotlin\com\esp32bridge\esp32_hid_bridge\HidController.kt
D:\Github\esp32_hid_bridge\android\app\src\main\AndroidManifest.xml
```

Build and install:

```bash
cd /d/Github/esp32_hid_bridge
flutter build apk --debug
adb -s 10.145.119.96:5555 install -r build/app/outputs/flutter-apk/app-debug.apk
adb -s 10.145.119.96:5555 shell monkey -p com.esp32bridge.esp32_hid_bridge 1 >/dev/null
```

Current package:

```text
com.esp32bridge.esp32_hid_bridge
```

Open app explicitly:

```bash
adb -s 10.145.119.96:5555 shell am start -n com.esp32bridge.esp32_hid_bridge/.MainActivity
```

Force restart app:

```bash
adb -s 10.145.119.96:5555 shell am force-stop com.esp32bridge.esp32_hid_bridge
adb -s 10.145.119.96:5555 shell monkey -p com.esp32bridge.esp32_hid_bridge 1 >/dev/null
```

## Confirm the phone is actually Bluetooth HID-connected to T25

Dump UI and look for status:

```bash
adb -s 10.145.119.96:5555 shell uiautomator dump >/dev/null
cmd //c "adb -s 10.145.119.96:5555 exec-out cat /sdcard/window_dump.xml" | grep -o "手機藍牙 HID[^\"]*"
```

Successful status looks like:

```text
手機藍牙 HID → T25
connect(T25/A8:41:F4:42:28:19)=true registered=true hid=true host=T25 A8:41:F4:42:28:19
```

The important parts are:

```text
registered=true
hid=true
connect(...)=true
```

If the app was freshly installed/restarted, it may first show `registered=false hid=false` while Android is handing over the HID profile; wait or press/register/connect again.

## What has been proven to work

The phone can act as a Bluetooth HID Device and T25 accepts it as keyboard/mouse once Windows Bluetooth is active.

Tested/implemented capabilities:

- Mouse move via HID report
- Left/right click
- Keyboard text typing for ASCII text
- `Enter`
- `Win+R`
- `Alt+Tab`
- `Ctrl+C`, `Ctrl+V`
- Opening a URL through the Windows Run dialog
- Opening Notepad and typing text, when the Windows side has focus and accepts the HID input

Log evidence for outgoing HID reports:

```bash
adb -s 10.145.119.96:5555 logcat -d -v time | grep -c "BTA_HdSendReport"
```

A nonzero count after pressing buttons means Android Bluetooth sent HID reports. It does **not** prove Windows executed the intended action because HID is practically one-way.

## HID is effectively one-way

For this use case, treat HID as one-way:

```text
phone HID Device → T25 Host
```

The phone can send:

- key presses
- mouse movement
- clicks
- scrolls

The phone cannot know via HID:

- what T25 screen currently shows
- whether Notepad/browser opened
- whether a command executed
- whether text landed in the correct field
- whether T25 entered BIOS

Strictly, HID can have tiny host-to-device feedback such as lock-key LEDs or feature reports, but this is not enough for screen/state awareness.

For reliable remote operation, pair HID input with a second feedback channel:

- RDP/VNC/AnyDesk/screen capture to see T25
- SSH/PowerShell to verify command execution
- camera pointed at T25 screen
- hardware KVM / PiKVM for BIOS-level feedback

## Current UI design guidance

The user objected to creating a special button for every task. Prefer a generic UI:

- Touchpad area
- Text field: send arbitrary keyboard text to T25
- Command/URL field: run arbitrary command/URL via `Win+R`
- Minimal connection/shortcut controls

Avoid adding one-off buttons like “Yahoo News”, “Notepad poem”, or “BIOS” unless it is for temporary debugging and then remove/simplify.

For ad-hoc tasks like “open Yahoo News and search重大新聞”, use the generic run-command/URL field or programmatically call `_openRunCommand(...)` instead of permanently adding another button.

Example URL for Yahoo News重大新聞:

```text
https://tw.news.yahoo.com/search?p=%E9%87%8D%E5%A4%A7%E6%96%B0%E8%81%9E
```

## USB cable from phone to T25 is not USB HID

When the phone is physically plugged into T25, verify USB state:

```bash
adb -s 10.145.119.96:5555 shell dumpsys usb | grep -E 'connected=|current_mode=|data_role=|power_role=|IsDeviceConnected|IsHostConnected|current_functions|kernel_state' | head -40
adb -s 10.145.119.96:5555 shell 'getprop sys.usb.config; getprop sys.usb.state'
```

Observed state:

```text
connected=true
current_mode=ufp
power_role=sink
data_role=device
kernel_state=CONFIGURED
sys.usb.config=mtp,adb
sys.usb.state=mtp,adb
```

Meaning:

```text
T25 = USB host
phone = USB device
USB function = MTP + ADB, not keyboard/mouse HID
```

So the cable is useful for charging/MTP/ADB, but **not** for BIOS keyboard/mouse control on stock Samsung firmware.

## BIOS / pre-boot control reality

Important distinction:

### Windows running / lock screen

Phone Bluetooth HID can control T25 when Windows Bluetooth stack is alive. It may work at lock/login screen after Windows has booted and restored Bluetooth HID.

Suggested test:

1. Ensure app status: `registered=true hid=true connect=true`
2. Lock Windows (`Win+L`)
3. Try phone touchpad/keyboard at lock screen
4. Try PIN/password + Enter if appropriate

### True BIOS / UEFI / boot menu

Stock Samsung phone Bluetooth HID usually **cannot** control true BIOS/UEFI because Windows Bluetooth stack is not running. T25 firmware would need native support for that Bluetooth HID device. Most PCs do not.

The USB cable currently does not help because it is MTP/ADB, not USB HID.

### Entering BIOS from Windows

Using phone HID while Windows is still running, the app can try to invoke:

```cmd
shutdown /r /fw /t 0
```

or stronger flow:

```text
Esc
Win+D
Win+R
cmd /c shutdown /r /fw /t 0
Enter
```

This only asks Windows to reboot into firmware setup. It does not guarantee the phone can control the firmware once Windows exits.

If T25 does not reboot after those HID sequences, the likely cause is on the Windows side: Run dialog not focused, command typed into the wrong window, command blocked/unsupported, or permissions.

## If the user insists on BIOS control

Be clear and concise:

- Phone Bluetooth HID controls Windows-level T25 after Windows Bluetooth is active.
- Stock Samsung USB is not HID.
- True BIOS control requires either:
  1. phone root + USB HID gadget support, risky and not guaranteed on Samsung A34, or
  2. a real USB HID/KVM device that BIOS sees.

Reliable BIOS-level architectures:

```text
phone → Wi-Fi/web/SSH → USB HID gadget device → T25 USB → BIOS
```

Possible USB HID devices:

- Raspberry Pi Zero / Zero 2 W in USB gadget HID mode
- Raspberry Pi Pico / RP2040 HID
- PiKVM
- commercial USB HID remote dongle

If the user says “不要 ESP32”, do not propose ESP32 as primary; propose Raspberry Pi Zero 2 W / RP2040 / PiKVM as alternatives.

## Useful ADB commands

Current foreground app:

```bash
adb -s 10.145.119.96:5555 shell dumpsys window | grep -E 'mCurrentFocus|mFocusedApp' | head -10
```

UI dump:

```bash
adb -s 10.145.119.96:5555 shell uiautomator dump >/dev/null
cmd //c "adb -s 10.145.119.96:5555 exec-out cat /sdcard/window_dump.xml" | head -c 12000
```

Find app buttons/bounds:

```bash
cmd //c "adb -s 10.145.119.96:5555 exec-out cat /sdcard/window_dump.xml" | grep -o "content-desc=\"[^\"]*\"[^>]*bounds=\"[^\"]*\""
```

Tap by coordinates:

```bash
adb -s 10.145.119.96:5555 shell input tap X Y
```

Swipe/scroll:

```bash
adb -s 10.145.119.96:5555 shell input swipe 800 1900 800 650 900
```

Clear and inspect Bluetooth HID sends:

```bash
adb -s 10.145.119.96:5555 logcat -c
# perform action
adb -s 10.145.119.96:5555 logcat -d -v time | grep -i -E "PhoneHid|BluetoothHid|HidDevice|BTA_HdSendReport|Exception|Security" | tail -120
```

## Code notes

`HidController.kt` uses Android `BluetoothHidDevice` with:

- composite keyboard + mouse HID descriptor
- report ID 1: boot-style keyboard report
- report ID 2: relative mouse report
- `BluetoothHidDevice.SUBCLASS1_COMBO`

The implemented method channel is:

```text
esp32bridge/hid
```

Supported methods from Flutter:

```dart
_hid('register', {'target': 'T25'});
_hid('connect', {'target': 'T25'});
_hid('status');
_hid('type', {'text': 'ASCII text'});
_hid('key', {'name': 'win+r'});
_hid('key', {'name': 'alt+tab'});
_hid('move', {'dx': 80, 'dy': 0, 'wheel': 0});
_hid('click', {'button': 1}); // 1 left, 2 right, 3 middle
```

Known key names include:

```text
enter, escape, backspace, tab, space, insert, delete, right, left, down, up,
home, end, pageup, pagedown, f1..f12
```

Modifiers:

```text
ctrl, shift, alt, win/gui/meta
```

Example combos:

```text
win+r
win+d
alt+tab
ctrl+c
ctrl+v
ctrl+alt+delete
```

## Communication style for this topic

The user values direct confirmation and dislikes unnecessary detours. Be explicit:

- “手機藍牙 HID 控 Windows：可以，已驗證。”
- “HID 是單向，手機不知道 T25 畫面結果。”
- “手機接 USB 到 T25 目前是 MTP/ADB，不是 USB HID，所以對 BIOS 沒用。”
- “真 BIOS 控制要 USB HID gadget/root or external HID/KVM.”

Avoid pretending HID can verify screen state. Always distinguish “sent HID reports” from “T25 actually did the intended thing.”

---

## Conformance Addendum

## When to Use
Use when controlling T25 from the Samsung phone via Bluetooth HID, maintaining the Flutter app D:\Github\esp32_hid_bridge that turns the phone into a Bluetooth keyboard/mouse, or reasoning about whether phone Bluetooth/USB can control T25 before Windows login or in BIOS. Covers ADB serials, app paths, tested capabilities, limitations of HID as a one-way channel, and the BIOS/USB-gadget constraints.

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
