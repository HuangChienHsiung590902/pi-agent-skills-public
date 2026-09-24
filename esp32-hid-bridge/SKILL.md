---
name: esp32-hid-bridge
description: Build, deploy, and troubleshoot the ESP32 HID Bridge system — an ESP32 emulating a Bluetooth HID keyboard/mouse, bridged to the network via a phone's Flutter app (USB-OTG host + background TCP relay + RFC2217 remote flashing), used to remotely control a target Windows PC (e.g. T25) that has no software installed on it at all. Use this skill whenever the user mentions ESP32 HID Bridge, esp32-ssh-bthid, esp32_hid_bridge, controlling T25 (or another target PC) via phone+ESP32, BLE HID keyboard/mouse commands (move/click/type/key), remote-flashing the ESP32 over the phone, or the phone app's background service / traffic panel / UI.
---

# ESP32 HID Bridge

## What this system actually is

Two separate repos work together to let an operator (or Codex) control a
target Windows PC that has **zero software installed on it** — the target
only needs its Bluetooth to be paired once with the ESP32 acting as a
keyboard/mouse:

```
Operator PC --(ZeroTier network)--> Phone (Flutter app) --(USB-OTG)--> ESP32 --(Bluetooth HID, wireless)--> Target PC (e.g. T25)
```

- **ESP32 firmware**: `C:\Users\HCH\esp32-ssh-bthid` — private GitHub repo
  `HuangChienHsiung590902/esp32-ssh-bthid`. ESP-IDF v5.3 project. Emulates a
  **BLE HID** combo keyboard+mouse (Classic BT HID does not interoperate
  reliably with Windows — this was tried first and abandoned; BLE HID works).
- **Phone bridge app**: `D:\Github\esp32_hid_bridge` — private GitHub repo
  `HuangChienHsiung590902/esp32_hid_bridge`. Flutter Android app, package
  `com.esp32bridge.esp32_hid_bridge`. Runs on the phone documented in the
  `phone-termux-remote` skill (Samsung Galaxy A34, ZeroTier IP
  `10.145.119.96`) — check that skill first for phone SSH/adb connectivity
  basics before touching this one.

**Why the phone+ESP32 combo and not something simpler**: a stock
(non-rooted) Android phone cannot itself act as a USB HID gadget (no root =
no USB peripheral/HID mode), **and** on this specific phone (Galaxy A34) its
Bluetooth controller also rejects registering as a Bluetooth **HID Device**
(peripheral) role — confirmed by a direct feasibility test via
`BluetoothHidDevice.registerApp()`, which returned `false` synchronously
(see "Bluetooth HID Device feasibility test" below). So the ESP32 is not
optional scaffolding — it is the only piece that can actually emulate a HID
device here. Do not suggest cutting it out unless a different, rooted, or
BLE-gadget-capable phone is involved.

**USB port is exclusive-or**: the phone's one USB-C port can be either a USB
**host** (talking to the ESP32 via OTG — the working setup) or a USB
**device** (plugged into a PC, e.g. for MTP/ADB) — never both at once. Do not
suggest "just plug the phone into the target PC directly" as a way to
control it; that path fundamentally cannot deliver keyboard/mouse input (see
above).

## Command protocol (phone app TCP port 5566)

Plain newline-terminated text commands, defined in
`components/cmd_parser/cmd_parser.c`:

```
move <dx> <dy>            relative mouse movement
click left|right|middle
type "<text>"             ASCII only, no Chinese/Unicode
key <name>                 e.g. key enter, key esc
key ctrl+<name>            modifier combos: ctrl/shift/alt/win(or gui), can chain e.g. ctrl+shift+l
```

Quick test client: `C:\Users\HCH\esp32-ssh-bthid\phone_cmd.py <cmd> [<cmd> ...]`
— connects to `10.145.119.96:5566`, sends each line, prints the real ESP32
response (not a synthetic ack — see gotcha below).

Typing into a **Windows lock/login screen** works fine over BLE HID (unlike
SSH-triggered UI automation, which cannot reach the secure Winlogon desktop)
— this is the correct tool specifically for that case. Sequence: `key enter`
(dismiss lock screen) → `type "<password>"` → `key enter`.

## Phone app architecture (background_service.dart + main.dart)

- All real state (USB port, TCP servers, traffic counters, local IPs) lives
  in `lib/background_service.dart`, running inside a
  `flutter_background_service` **foreground service** (survives the app
  being backgrounded/screen off). `lib/main.dart` is a thin UI mirror synced
  via `invoke`/`on` — never put real logic in main.dart's State class.
- Ports: **5566** = command relay (forwards text lines to the ESP32 over USB
  serial, relays the ESP32's real serial output back — not a fake "ok").
  **5577** = `Rfc2217Server` (custom from-scratch RFC2217 implementation in
  `lib/rfc2217_server.dart`) for remote `idf.py flash` over the network.
- **Multi-client is supported by design**: the command server holds
  connected operators in `final Set<Socket> _operatorSockets = {}`
  (`background_service.dart`) — every accepted socket is added to the set,
  any socket's incoming line gets written to the shared ESP32 UART, and the
  ESP32's serial replies are broadcast to *every* socket in the set. So
  multiple operators (multiple scripts, multiple AI agent panes, etc.) can
  hold connections to port 5566 at the same time and all send commands — they
  share one physical mouse/keyboard cursor on the target PC since it's one
  UART, but the server itself does not reject or serialize extra clients.
  Verified live 2026-08-17: 10 concurrent short-lived connections all
  succeeded (`ok=10 fail=0`), and two separate real AI agent panes (a Codex
  Code pane and a pi/codex pane) each independently connected and sent
  commands successfully.
- USB device labeling: only VID `0x10C4` (CP2102, the ESP32's onboard
  USB-UART chip) is confidently labeled "ESP32". Anything else is labeled
  generically "已連線" (connected) — **do not** label it "電腦" (computer);
  a plain computer plugged in via USB does not even show up in
  `UsbSerial.listDevices()` in the first place (it doesn't present as a
  USB-serial device), so that code path is for some other serial adapter,
  not a general "phone talking to a PC" detector.
- Status tab also lists every local IPv4 address the operator PC can
  connect to (`iface: address`, from `NetworkInterface.list()` in
  `_loadLocalIps()`), each shown with the command port (5566) appended in
  the UI — pushed via the `ips` service event and included in `fullState`.
- Traffic panel shows **real device network usage** (Android `TrafficStats`,
  mobile+WiFi combined, via a native `esp32bridge/traffic` MethodChannel in
  `MainActivity.kt`) — not our own relay-protocol byte counts. If asked for
  "流量" in this app's UI, this is almost certainly what's meant.
- UI theme: flat/restrained, explicitly **not** the gradient+glow "AI
  generated" look for the *UI chrome itself* (cards, buttons, backgrounds).
  Light palette: bg `#F5F6F8`, cards `#FFFFFF`, text `#1B242C`/`#6B7684`,
  accent `#2E8B87`. (A dark variant exists in git history if ever needed: bg
  `#0F1720`, cards `#16212B`, text `#D7DEE5`/`#8A97A3`, same accent-ish teal
  `#6FB7B7`.) The **app icon** is the one exception allowed to be colorful —
  after iterating through a flat single-tone version and two alternative
  concepts (see below), the user picked a colorful "remote-control /
  signal-flow" design: phone (with a D-pad hint) → colorful command-stream
  arc → green ESP32 chip (labelled "32") → Bluetooth symbol → computer
  screen (cursor + keyboard hint), plus a status/traffic/log icon strip at
  the bottom. Source SVG is `assets/icon/app_icon.svg`; regenerate via
  `flutter_launcher_icons` (see Build below). The two concepts that were
  generated but *not* chosen (kept in git history/`design/` for reference if
  the user ever wants to switch back) were: (A) a minimalist geometric
  bridge — abstract phone/computer blocks joined by a single curved gradient
  bridge line through a central hub chip, and (C) the original literal
  phone→ESP32(labelled "ESP")→Bluetooth→computer illustration. If asked to
  regenerate icon concepts again, keep them in this same colorful-but-clean
  style (no heavy glow/blur filters — soft drop-shadows only) and always
  produce 2-3 distinctly different concepts rather than committing to one.
- **Login page + bottom tabs**: `lib/login_page.dart` is a UI-only gate (no
  credential validation against anything — any input or blank proceeds) that
  `pushReplacement`s into `BridgeHomePage`; shown at 160×160 with a rounded
  clip. `BridgeHomePage` (`main.dart`) has three bottom `NavigationDestination`s
  — 狀態 (status card + IP list + reconnect/refresh buttons), 流量 (traffic
  card), LOG (dark console) — switched via `IndexedStack` so tab state
  survives switching. An **exit button** (power icon, top-right of the
  AppBar) shows a confirm dialog, then calls `_service.invoke('stopService')`
  (which the background service already handles — cancels the traffic timer,
  closes the USB port/TCP servers/flash server, then `service.stopSelf()`)
  followed by `SystemNavigator.pop()` to fully quit, as opposed to just
  backgrounding the app (which deliberately keeps the service alive).

## Build & deploy (phone app)

```bash
export PATH="/d/flutter/bin:$PATH"
export JAVA_HOME="/c/Program Files/Eclipse Adoptium/jdk-17.0.20.8-hotspot"
export ANDROID_HOME="$HOME/AppData/Local/Android/Sdk"
cd /d/Github/esp32_hid_bridge
flutter analyze lib/          # always clean before building
flutter build apk --debug
```

Regenerate launcher icons after editing `assets/icon/app_icon.svg` (needs
`npm install --no-save sharp` once, in-repo, gitignored):
```bash
node -e "require('sharp')('assets/icon/app_icon.svg',{density:384}).resize(1024,1024).png().toFile('assets/icon/app_icon.png')"
dart run flutter_launcher_icons
```

Deploy to the phone (ZeroTier IP, see `phone-termux-remote` skill):
```bash
export PATH="/d/BIN/platform-tools:$PATH"
export MSYS_NO_PATHCONV=1
adb -s 10.145.119.96:5555 install -r "D:/Github/esp32_hid_bridge/build/app/outputs/flutter-apk/app-debug.apk"
adb -s 10.145.119.96:5555 shell am force-stop com.esp32bridge.esp32_hid_bridge
adb -s 10.145.119.96:5555 shell am start -n com.esp32bridge.esp32_hid_bridge/.MainActivity
```
If a screen looks stale after install, **force-stop explicitly and wait for
the process list to actually be empty** before relaunching — installs can
race with a still-running old process and silently reuse its state.

Reinstall from scratch (not just `-r`) resets granted permissions
(notifications, Bluetooth, etc.) and will re-trigger the Android permission
dialogs — expect and handle that after `adb uninstall` + `adb install`.

## ESP32 firmware: build & flash

```powershell
. C:\Espressif\esp-idf\export.ps1   # PowerShell only - MSYS/Git-Bash is unsupported by idf.py
cd C:\Users\HCH\esp32-ssh-bthid
idf.py build
idf.py -p COM6 flash                                   # direct USB (ESP32 on this PC)
idf.py -p "rfc2217://10.145.119.96:5577" flash          # remote, via the phone (no ign_set_control - see gotcha)
```

## ChatGPT / MCP connector mode (2026-08)

The phone app now also exposes a **Streamable HTTP MCP server** for ChatGPT
connectors, in addition to the raw TCP HID relay on port 5566.

Important constants in `D:\Github\esp32_hid_bridge\lib\background_service.dart`:

```dart
const int mcpPort = 5588;
const String mcpAuthToken = '<KB_BEARER_TOKEN>';
```

Authentication accepts either:

- Header form for normal clients:
  `Authorization: Bearer <KB_BEARER_TOKEN>...` style, specifically
  `Bearer <KB_BEARER_TOKEN>`.
- **Path-token form** for ChatGPT connectors, because the ChatGPT connector
  UI only offered `OAuth` / `無驗證` / `混合` and had no custom bearer-header
  field. The Dart helper `_isAuthorizedMcpRequest(HttpRequest request)`
  accepts requests when `request.uri.pathSegments.contains(mcpAuthToken)`.

Current public connector URL:

```text
https://hid.james-huang.org/mcp/<KB_BEARER_TOKEN>
```

Cloudflare tunnel route (see `cloudflare-tunnel` skill) forwards:

```text
hid.james-huang.org -> http://10.145.119.96:5588
```

When configuring ChatGPT Desktop/Web connector:

```text
Name: ESP32 HID Bridge
Description: 透過 BLE 控制電腦的鍵盤與滑鼠 (ESP32-S3 HID)
URL: https://hid.james-huang.org/mcp/<KB_BEARER_TOKEN>
Authentication: 無驗證
```

After changing MCP tools or descriptions, ChatGPT may cache the old tool
schema. Open connector settings and click `重新整理` (refresh), or start a new
chat, before concluding a tool is missing.

Security note: the path token is effectively a password embedded in a public
URL. If it is exposed beyond this operator setup, rotate
`mcpAuthToken` in `background_service.dart`, rebuild/reinstall the APK, and
update the ChatGPT connector URL.

### MCP tools exposed by the phone app

ChatGPT connector currently shows these public actions (verified from the
connector's Development-mode app page on 2026-08-21, version ID
`asdk_app_v_6a87beabdbe481918b92d203d17a208c`, app ID
`asdk_app_6a87beabdbd48191931e2eaf6031bdb0`, URL exactly the path-token URL
above, auth = none / 無):

Low-level tools:

- `hid_type` — input `{ "text": string }`; types text on the target.
- `hid_key` — input `{ "keys": string }`; examples: `enter`, `escape`,
  `ctrl+alt+delete`, `win+r`, `ctrl+shift+l`.
- `hid_move` — input `{ "dx": integer, "dy": integer }`; relative mouse
  movement.
- `hid_click` — input `{ "button": "left"|"right"|"middle" }`.
- `hid_scroll` — input `{ "amount": integer }`; positive up, negative down.
- `hid_status` — input `{}`; reports whether ESP32-S3 BLE HID is connected.

High-level command-line tools added in `background_service.dart`:

- `hid_open_command_line` — input `{ "shell"?: "powershell"|"cmd"|"wt", "as_admin"?: boolean }`;
  opens Windows Run → chosen shell → Enter. `as_admin=true` uses Ctrl+Shift+Enter
  and can trigger UAC that HID cannot answer without visual/user help.
- `hid_type_command` — input `{ "command": string, "press_enter"?: boolean }`;
  types a command into the already-focused terminal.
- `hid_run_command` — input `{ "command": string, "shell"?: "powershell"|"cmd"|"wt", "open_shell"?: boolean, "press_enter"?: boolean }`;
  optionally opens a shell, types the command, optionally presses Enter.
- `hid_batch` — input `{ "steps": [ ... ] }`; each step has `action` one of
  `type`, `key`, `move`, `click`, `scroll`, `delay`; step fields are:
  `text`, `keys`, `dx`, `dy`, `button`, `amount`, `delay_ms` as appropriate.

ChatGPT labels these tools with `建議定義輸出結構` because the MCP actions do not
currently define an explicit structured output schema. That warning is not a
runtime failure; all actions return text like `sent: ...`.

Helpers added for these tools:

- `_sendBleCommandAndWait`
- `_shellLaunchCommand`
- `_openCommandLine`
- `_typeCommandInTerminal`
- `_runCommandViaHid`
- `_runMcpBatch`

The high-level tool descriptions deliberately remind ChatGPT that **HID is
input-only**: it can send keyboard/mouse actions, but cannot read screen or
stdout. For output/verification it must ask the user to open the ESP32 HID
Bridge App live camera / AI snapshot, or ask for a photo.

Quick MCP sanity tests:

```bash
# token in path, no bearer header
curl -s https://hid.james-huang.org/mcp/<KB_BEARER_TOKEN> \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  --data '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"curl","version":"1"}}}'
```

If calling through an MCP client, verify the tool list contains BOTH readback
Agent tools and HID tools. As of 2026-08-21 the Agent aliases are intentionally
listed **first** because ChatGPT sometimes appeared to cache/truncate the tool
schema after seeing only the HID actions:

```text
agent_status, agent_screenshot, agent_ocr_screen,
agent_run_command_capture, agent_read_text_file, agent_write_text_file,
hid_type, hid_key, hid_move, hid_click, hid_scroll,
hid_open_command_line, hid_type_command, hid_run_command, hid_batch, hid_status,
win_bridge_status, win_bridge_deploy_via_hid, win_screenshot, win_ocr_screen,
win_run_command_capture, win_read_text_file, win_write_text_file
```

A current `tools/list` smoke test should start with:

```text
['agent_status', 'agent_screenshot', 'agent_ocr_screen',
 'agent_run_command_capture', 'agent_read_text_file', 'agent_write_text_file', ...]
```

Old known-good `hid_run_command` logs may mention `COMBO GUI+R`; do **not**
copy that for deployment anymore. For deployment/open-shell flows the app now
uses `CTRL+ESC` → type `powershell` → `ENTER`, and types long commands in
~48-character chunks. This avoids a prior target-side failure where a malformed
Win/Shift/letter combo opened Windows Snipping Tool instead of Run/PowerShell.

Long HID typed commands must be chunked with whitespace-aware boundaries. The
ESP32 firmware trims each received command line before `Keyboard.print(rest)`,
so a chunk ending in a space, or the next chunk beginning with a space, loses
that separator. This already broke deployment once by producing invalid
PowerShell such as `iwrhttp://...` and `$p--phone`. The phone app fix is
`_hidTypeChunks(...)` in both `main.dart` and `background_service.dart`, which
extends each chunk until the boundary is not adjacent to whitespace. Preserve
that logic when editing deploy/HID typing code.

## Windows Bridge Agent / direct readback mode (方案 B)

The phone app is now the single ChatGPT MCP gateway. It routes:

```text
hid_* / hid_* high-level tools -> BLE/ESP32 HID -> target PC keyboard/mouse
agent_* / win_* tools          -> WindowsBridgeAgent on target PC -> readback
```

Ports:

```text
5588 = ChatGPT MCP server
5589 = phone camera AI snapshot JPEG endpoint
5590 = Windows Bridge Agent WebSocket relay + WindowsBridgeAgent.exe download
```

Windows agent source and bundled executable:

```text
D:\Github\esp32_hid_bridge\windows_bridge_agent\Program.cs
D:\Github\esp32_hid_bridge\windows_bridge_agent\WindowsBridgeAgent.csproj
D:\Github\esp32_hid_bridge\assets\windows_bridge\WindowsBridgeAgent.exe
```

The bundled EXE is a .NET 8 self-contained single-file `win-x64` console app
(~65MB; GitHub warns about >50MB but current repo push succeeds). Rebuild it:

```bash
cd /d/Github/esp32_hid_bridge/windows_bridge_agent
dotnet publish WindowsBridgeAgent.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:PublishTrimmed=true -o ../assets/windows_bridge
cd ..
flutter build apk --debug
```

The phone app serves the staged EXE at:

```text
http://10.145.119.96:5590/WindowsBridgeAgent.exe
```

Agent connection URL:

```text
ws://10.145.119.96:5590/bridge/ws
```

### Deploy/redeploy Agent to the target PC

UI path:

```text
Win橋接 tab -> 啟動部署服務 -> 用 HID 部署
```

MCP path:

```text
win_bridge_deploy_via_hid
```

The deployment command typed into PowerShell is deliberately short and runs the
agent in the same visible PowerShell window (so SmartScreen/network errors are
visible). As of commit `9dbfb48`, deployment is **reuse-first**: it checks
`%TEMP%\wba.exe` on the target PC and only downloads from the phone when that
file is missing. This prevents every `用 HID 部署` click from re-downloading the
65MB EXE. As of commit `6e85284`, the command also guards the final execution:
if the conditional download fails, it prints an explicit error instead of trying
to execute a nonexistent `%TEMP%\wba.exe`:

```powershell
$p="$env:TEMP\wba.exe"; if(!(Test-Path $p)){iwr http://10.145.119.96:5590/WindowsBridgeAgent.exe -OutFile $p}; if(Test-Path $p){& $p --phone ws://10.145.119.96:5590/bridge/ws --token <KB_BEARER_TOKEN>}else{Write-Error 'WindowsBridgeAgent download failed'}
```

If `agent_status` reports `agentVersion: 0.1.0`, OCR is NOT available yet.
Because deployment now reuses `%TEMP%\wba.exe`, a normal redeploy may keep
running the old binary. To force-update the target agent, delete the target
file first, then deploy again:

```powershell
Remove-Item "$env:TEMP\wba.exe" -Force -ErrorAction SilentlyContinue
```

or add/use a future explicit "force redownload" button/tool. Old agents will
answer `unknown tool: ocr_screen` when `agent_ocr_screen` is called.

If deployment shows PowerShell `InvalidOperation ... HttpWebRequest ...
Invoke-WebRequest` on the `iwr http://10.145.119.96:5590/WindowsBridgeAgent.exe
-OutFile $p` line, the target PC failed to download the EXE from the phone.
First confirm the phone is serving it from another machine:

```bash
curl -I http://10.145.119.96:5590/WindowsBridgeAgent.exe
# expected: 200 OK, and the body starts with MZ for a PE executable
```

Then diagnose target-to-phone network reachability from the target PowerShell:

```powershell
Test-NetConnection 10.145.119.96 -Port 5590
Invoke-WebRequest http://10.145.119.96:5590/WindowsBridgeAgent.exe -OutFile $env:TEMP\wba.exe
```

A failed download usually means the target PC cannot reach the phone's ZeroTier
IP/port 5590 at that moment (different network, ZeroTier/firewall issue, app
service not running, or Android killed/restarted the service). Do not mistake
this for an Agent bug.

### Agent tools and what to use when

Canonical public aliases for ChatGPT:

- `agent_status` — reports online/offline, computerName (e.g. `T25`), user,
  OS, version, lastSeen.
- `agent_run_command_capture` — runs PowerShell/cmd on the target and returns
  `stdout/stderr/exitCode`. Fastest for hardware specs, processes, files,
  network checks, etc.
- `agent_read_text_file` / `agent_write_text_file` — read/write UTF-8 text on
  the target.
- `agent_screenshot` — captures the primary screen and returns MCP image
  content (`image/jpeg`). Useful when the agent needs to visually inspect the
  screen, but heavier than OCR/text.
- `agent_ocr_screen` — captures primary screen, runs built-in Windows
  `Windows.Media.Ocr` on the target, returns `fullText` plus per-word
  `{text,x,y,width,height}`. This is the best feedback path for UI automation:
  much lighter than screenshots and gives coordinates for `hid_move`/`hid_click`.

`win_*` names are equivalent lower-level aliases: `win_bridge_status`,
`win_screenshot`, `win_ocr_screen`, `win_run_command_capture`,
`win_read_text_file`, `win_write_text_file`.

Three feedback/readback methods, in preference order:

| Method | Software on target? | Best use | Notes |
|---|---:|---|---|
| Phone photo / AI snapshot | No | Fast visual confirmation | User must aim camera; fastest for human-in-loop. |
| Windows Agent command/OCR | Yes | UI automation, reading text, diagnostics | `agent_ocr_screen` + HID = eyes + hands. |
| Windows Agent screenshot | Yes | Full visual inspection | Heavier; use only when OCR/text is insufficient. |

### OCR output shape

`agent_ocr_screen` accepts:

```json
{"language":"zh-Hant-TW", "max_items":200, "timeout_seconds":30}
```

Returns text JSON like:

```json
{
  "ok": true,
  "width": 1920,
  "height": 1080,
  "language": "zh-Hant-TW",
  "fullText": "...",
  "items": [
    {"text":"PowerShell", "x":100, "y":50, "width":120, "height":24}
  ]
}
```

Use item centers for HID clicking: `click_x = x + width/2`,
`click_y = y + height/2`.

## AI snapshot / visual feedback camera

The phone app has a camera/AI snapshot page in:

```text
D:\Github\esp32_hid_bridge\lib\features\camera\ai_snapshot_page.dart
```

It serves the current camera frame as JPEG at:

```dart
const int aiSnapshotPort = 5589;
```

Keep this port **different** from the MCP server port 5588. A previous bug had
AI snapshot also trying to bind `0.0.0.0:5588`, causing the phone UI error:

```text
Failed to create server socket
address = 0.0.0.0, port = 5588
```

Current snapshot URL on the phone's ZeroTier IP:

```text
http://10.145.119.96:5589/snapshot
```

Use this to provide visual feedback to ChatGPT or to the operator after HID
commands open something on the target PC. If the snapshot shows fabric/desk
instead of the target screen, do **not** infer command output; ask the user to
aim the phone camera at the HID-controlled target computer screen and retry.

Example local check:

```bash
curl -I http://10.145.119.96:5589/snapshot
# expected: 200 image/jpeg
```

## Control-only vs readback: ESP32 HID is hands, not eyes

A critical limitation to always preserve in prompts and designs:

```text
ESP32 HID = hands
Windows Bridge / MCP Agent / camera = eyes and ears
```

With **only** ESP32 HID and no software on the target PC, the system can:

| Can do | Cannot do |
|---|---|
| simulate keyboard input | read screen text |
| simulate mouse movement/clicks | capture screenshots |
| open programs and type commands | read file contents |
| make the target display results | automatically return results to ChatGPT |

So there are three operating modes:

| Mode | Need software on controlled PC? | Capability |
|---|---:|---|
| ESP32 HID only | No | control only; no readback |
| ESP32 HID + phone camera/photo | No | control + visual readback through user/phone camera |
| ESP32 HID + Windows Bridge/MCP Agent | Yes | control + direct readback |

If the user wants a full bidirectional version, use the built-in方案 B Windows
Bridge Agent described above. It exposes `agent_run_command_capture`,
`agent_read_text_file`, `agent_screenshot`, and `agent_ocr_screen` through the
same phone MCP connector. That is a different trust model: it violates the
original "target PC has zero software installed" constraint. For the
zero-install path, always use Notepad / visible UI output plus phone camera or
AI snapshot for readback.

## Correct ChatGPT delegation pattern

When the user says ChatGPT should do the operation, do **not** directly call
the HID MCP from this operator agent unless explicitly asked. The intended
final chain is:

```text
Phone/ChatGPT Desktop ChatGPT
  -> ESP32 HID Bridge MCP connector
  -> phone app MCP server
  -> BLE/USB/ESP32 HID
  -> target Windows PC

Phone camera / AI snapshot
  -> ChatGPT/operator reads target screen visually
```

So this agent's role is usually to configure/build/debug the bridge and then
send a task prompt to ChatGPT. ChatGPT should call the connector tools itself.

Prompt template to give ChatGPT for collecting target-PC hardware specs:

```text
你現在要透過 ESP32 HID Bridge 外掛控制 HID 目標電腦，查詢「那台被 ESP32 HID 控制的電腦」硬體規格。不要查你自己所在的 ChatGPT 環境，也不要假設能直接讀取資料；所有操作都要透過 ESP32 HID Bridge 外掛送鍵盤/滑鼠到目標電腦。

請照這個流程做：
1. 呼叫 hid_status，確認回傳 connected to ESP32-S3-HID。
2. 呼叫 hid_run_command，在 HID 目標電腦開 cmd，並執行下列命令：

(echo ==== COMPUTER ==== & hostname & echo ==== SYSTEM ==== & wmic computersystem get manufacturer,model,totalphysicalmemory & echo ==== CPU ==== & wmic cpu get name,numberofcores,numberoflogicalprocessors & echo ==== GPU ==== & wmic path win32_VideoController get name,AdapterRAM & echo ==== DISK ==== & wmic diskdrive get model,size & echo ==== OS ==== & ver) > %TEMP%\hid_specs.txt & notepad %TEMP%\hid_specs.txt

參數請用：shell=cmd, open_shell=true, press_enter=true。
3. 執行後，如果你需要看結果，請要求我打開 ESP32 HID Bridge App 的 AI 快照/即時影像或傳螢幕照片給你。
4. 看到 Notepad 裡的結果後，請整理成繁體中文硬體規格表。
```

Target-hardware collection must mean the **HID-controlled target computer**,
not this operator laptop. A previously mistaken local/operator result was
`GRAM / LG Electronics / 16T90TP-K.AD78C2 / Intel Core Ultra 7 255H`; do not
use that as the target result.

If `wmic` is unavailable on the target, rerun through PowerShell/CIM and still
write/open a text file in Notepad so the camera can read it:

```powershell
$lines = @()
$cs = Get-CimInstance Win32_ComputerSystem
$cpu = Get-CimInstance Win32_Processor
$gpu = Get-CimInstance Win32_VideoController
$disk = Get-CimInstance Win32_DiskDrive
$os = Get-CimInstance Win32_OperatingSystem
$lines += '==== COMPUTER ===='; $lines += $env:COMPUTERNAME
$lines += '==== SYSTEM ===='; $lines += "Manufacturer=$($cs.Manufacturer) Model=$($cs.Model) TotalPhysicalMemory=$($cs.TotalPhysicalMemory)"
$lines += '==== CPU ===='; $lines += ($cpu | Select-Object Name,NumberOfCores,NumberOfLogicalProcessors | Out-String)
$lines += '==== GPU ===='; $lines += ($gpu | Select-Object Name,AdapterRAM | Out-String)
$lines += '==== DISK ===='; $lines += ($disk | Select-Object Model,Size | Out-String)
$lines += '==== OS ===='; $lines += "Caption=$($os.Caption) Version=$($os.Version) Build=$($os.BuildNumber)"
$lines | Set-Content $env:TEMP\hid_specs.txt -Encoding UTF8
notepad $env:TEMP\hid_specs.txt
```

## Sending prompts into ChatGPT Desktop/Web

Preferred ways, in order:

1. If Chrome/ChatGPT Web is open with CDP port 9222, attach to it (see
   `connect-chrome`) and inject the prompt into `https://chatgpt.com/`.
2. If controlling the Android phone, ensure ADB sees the phone (`RFCW3049BHM`
   or `10.145.119.96:5555`) and send text to the phone ChatGPT UI.
3. If neither is connected, give the prompt to the user to paste manually.

Do not claim the prompt was sent unless one of those channels actually
succeeds. Failure modes already hit:

```text
Chrome CDP: connect ECONNREFUSED 127.0.0.1:9222
ADB: device 'RFCW3049BHM' not found
```

For ChatGPT Desktop window screenshots/control, temporary helper scripts were
used during investigation:

```text
C:\Users\HCH\AppData\Local\Temp\capture_chatgpt_desktop.ps1
C:\Users\HCH\AppData\Local\Temp\chatgpt_click_back.ps1
C:\Users\HCH\AppData\Local\Temp\chatgpt_send_prompt.ps1
```

They are disposable session helpers, not canonical project code.

## Known gotchas (all previously hit — don't rediscover)

1. **RFC2217 remote flashing is STILL BROKEN (open problem, actively
   investigated 2026-08-18, not solved) — `Wrong boot mode detected (0x13)`
   on every attempt so far despite multiple fixes.** Read this whole entry
   before touching `rfc2217_server.dart` or the ESP32 repo's reset config
   again; a lot has already been tried and ruled out.

   **Command to reproduce** (PowerShell only, MSYS/Git Bash's `export.ps1`
   will refuse to run unless you first `Remove-Item Env:MSYSTEM,
   Env:MINGW_PREFIX, Env:MSYS, Env:OSTYPE` when invoking PowerShell from a
   Bash-based agent):
   ```powershell
   . C:\Espressif\esp-idf\export.ps1
   cd C:\Users\HCH\esp32-ssh-bthid
   idf.py -p "rfc2217://10.145.119.96:5577" flash
   ```

   **Root cause, confirmed by reading esptool/pyserial source directly**
   (`esptool/loader.py`, `esptool/reset.py`, `serial/rfc2217.py`) — not
   guessed:
   - `rfc2217://` targets on Windows are forced through esptool's
     `ClassicReset`, which drives the ESP32 into download mode using two
     hardcoded absolute `time.sleep()` calls to hold a specific ordering
     between the EN (reset, driven by RTS) and IO0 (boot-strap, driven by
     DTR) pins: `setRTS(True) → sleep(0.1) → setDTR(True) → setRTS(False) →
     sleep(reset_delay) → setDTR(False)`.
   - Under `rfc2217://`, every single `setDTR`/`setRTS` call is a
     **synchronous blocking network round-trip**: pyserial's
     `rfc2217_set_control()` calls `item.wait(timeout)`, which itself polls
     in `time.sleep(0.05)` (50ms) ticks waiting for our echo ACK — on top of
     that, our Android `usb_serial` control transfer itself measured at
     **85-162ms** real latency to actually reach the CP2102.
   - esptool assumes `setDTR(True)` (IO0→LOW) and `setRTS(False)` (EN→HIGH,
     chip released) happen at nearly the same instant with no sleep between
     them. Over RFC2217, each of those calls independently blocks for
     50-160ms with independent, *inconsistent* latency — so occasionally the
     EN-release USB transfer physically lands before the IO0-LOW transfer
     has actually taken effect on the chip, even though our software issued
     them in the right order and awaited each one. Chip releases from reset
     with IO0 still HIGH → boots normally instead of into the ROM loader →
     `0x13`. This is exactly why the failure is **intermittent**, not
     constant.
   - This means the earlier "await before echoing" fix (see git history) is
     real and necessary but **not sufficient** — awaiting only guarantees
     Android's USB stack *accepted* the control transfer request, not that
     the CP2102 has actually finished driving the GPIO line.

   **Tried and confirmed NOT sufficient on their own:**
   - Serializing RFC2217 subnegotiation handling in `rfc2217_server.dart`
     via a per-client `Future` queue (a real correctness bug existed
     separately — `handleSubnegotiation()` was called without awaiting
     inside a synchronous `socket.listen` callback, letting multiple
     control transfers go in-flight concurrently — worth keeping this fix
     regardless, but it did not fix the boot-mode failure by itself).
   - An ESP32-side `esptool.cfg` with `custom_reset_sequence` inserting an
     extra `W0.25`-`W0.3` wait between the `D1` (IO0→LOW) and `R0` (EN→HIGH)
     steps, in two different tried durations. `esptool.cfg` in the ESP32
     repo root IS correctly auto-discovered by `idf.py flash` (confirmed via
     `config.py:load_config_file()`'s search order — cwd first) and IS
     accepted by esptool (confirmed via `ESPTOOL_CFGFILE` env var forcing a
     "Loaded custom configuration from ..." log line) — the mechanism works,
     it just didn't fix the underlying failure.

   **Newest, more concerning finding (not yet resolved) — RTS may not
   reliably control IO0 on this Android/CP2102 path at all:** direct pyserial
   probes toggling DTR/RTS independently over the RFC2217 link found DTR
   alone reliably resets the chip (clean `POWERON_RESET` boot log every
   time), but no combination of RTS transitions produced an observable
   download-mode boot log — only ever `boot:0x13 (SPI_FAST_FLASH_BOOT)`
   (normal boot). This suggests the problem might not be purely timing —
   RTS might not be wired to / effectively controlling this board's IO0/BOOT
   strap at all through this phone's `usb_serial`+CP2102 path (wiring,
   polarity, or driver-level issue), which no amount of reset-sequence
   timing tuning would fix.

   **Next diagnostic step (not yet done):** flash this exact ESP32 board via
   a **direct USB connection to a PC** (bypassing the phone/RFC2217 entirely)
   and see if `idf.py -p COM6 flash` auto-enters download mode reliably. If
   direct-USB also fails to auto-reset, the problem is with this board's
   auto-reset wiring/circuit in general, not RFC2217-specific. If direct-USB
   works fine, the problem is specific to the Android `usb_serial` control
   line path (CP2102 modem-control implementation, e.g.
   `com.felhr.usbserial.CP2102SerialDevice.setDTR/setRTS` sending
   `0x0101`/`0x0202` MHS requests) and RFC2217 latency on top of it.
   **Do not report this as fixed until multiple consecutive `idf.py flash`
   runs succeed** — a single success is not enough given how intermittent
   this failure is.

   Historical note: `?ign_set_control` in the flash URL makes esptool's
   client **not wait for our ack at all** (flat 100ms sleep instead), which
   defeats the synchronization the "await before echo" fix relies on — omit
   it so the client genuinely synchronizes on our echo. This was already
   confirmed correct and is not the open part of the problem.

   Full investigation log (source-code citations, every tried config,
   probe results) is in `D:\Github\esp32_hid_bridge\design\rfc2217_collab_notes.md`
   — check there before re-deriving any of the above from scratch.
2. **HID report byte-length must exactly match the report descriptor.** A
   descriptor declaring `Report Count 5` for the keycode array but C code
   sending an 8-byte buffer (implying 6 keys) causes Windows to silently
   discard every report of that type while accepting others whose length
   happens to match. If mouse works but keyboard doesn't (or vice versa),
   check this first, in both `bt_hid_combo.c` and the report map bytes.
3. **After changing the BLE report descriptor, Windows keeps serving the
   stale cached one from the original pairing (GATT Service Changed
   caching).** Remove and re-pair the Bluetooth device on the target PC
   after any descriptor change, or it'll look "half broken" (connects fine,
   reports silently ignored).
4. **Android 13+ notification permission must be requested BEFORE
   `service.configure()`/start, not lazily.** Without it,
   `startForeground()` inside the service fails silently, and Android kills
   the process a few seconds later with
   `ForegroundServiceDidNotStartInTimeException` — looks like a random
   background-only crash loop. Fixed by calling
   `requestNotificationsPermission()` in `initializeService()` in
   `background_service.dart` before `service.configure()`. Also wrap any
   notification-update code in try/catch — a failure there must never be
   able to abort the TCP servers' startup sequence (this exact bug hid
   behind an unrelated `invalid_icon` PlatformException once).
5. **Opening a local serial port resets the ESP32** (pyserial and
   `usb_serial` both toggle DTR/RTS on open by default). Any script talking
   to the ESP32 fresh needs a `time.sleep(4)`-ish wait after opening before
   sending commands, to let BLE reconnect.
6. **Phone's wireless adb / ZeroTier link can drop intermittently** even
   without a reboot — `ping` failing with high loss to `10.145.119.96` and
   the app's own TCP ports (5566) also timing out at the same time means
   the phone genuinely fell off network (not an app/adb-specific bug); nothing
   fixable from the PC side, needs checking the phone screen / ZeroTier
   toggle directly. If only `adb` (port 5555) is refused but ping/5566 both
   work, that's `adb tcpip` mode resetting — see `phone-termux-remote` skill
   Gotcha #1 (needs USB re-trigger, which conflicts with the ESP32
   occupying the phone's only USB port — there is no remote fix for this
   specific combination).
7. **Bluetooth HID Device feasibility test (Galaxy A34): unsupported.**
   Tested directly via `BluetoothAdapter.getProfileProxy(..., HID_DEVICE)` →
   proxy obtained fine → `BluetoothHidDevice.registerApp(sdp, qos, qos,
   executor, callback)` → returned `false` **synchronously** (rejected by
   the Bluetooth stack/HAL before even reaching the async callback). This
   means this phone cannot act as a Bluetooth HID peripheral itself, at all
   — confirms gotcha zero above. If a different phone model is ever
   substituted, this specific test (small `MethodChannel` in
   `MainActivity.kt` calling `registerApp` with a standard boot-keyboard
   descriptor) is the fast way to re-check, before investing in building
   full HID-over-Bluetooth support into the app.
8. **USB device auto-permission**: `android/app/src/main/res/xml/device_filter.xml`
   (VID `4292` = `0x10C4`) + a `USB_DEVICE_ATTACHED` intent-filter on
   `MainActivity` auto-grants USB permission when the ESP32 is plugged in,
   without a dialog — needed because the background service can't show a
   permission dialog itself if the ESP32 gets unplugged/replugged while the
   UI isn't in the foreground.
9. **`_operatorSockets` used to leak on abnormal disconnect (FIXED
   2026-08-17, commit `aefac2d`).** In `_startServer()`
   (`background_service.dart`), a socket was only removed from
   `_operatorSockets` inside `socket.done.then((_) { ... remove ... })`. A
   stress test (10 concurrent short-lived connections) left the LOG full of
   `SocketException: Connection reset by peer (errno=104)` and the "連線數"
   (active connections) counter permanently stuck too high (14, should have
   returned to ~0) — an abrupt client-side close (TCP RST, not a clean FIN)
   fires the inner `socket.listen(..., onError: ...)` handler but does not
   reliably complete the `socket.done` future, so the socket reference never
   got removed from the set. Fixed by also calling `_operatorSockets.remove
   (socket)` + `socket.destroy()` inside that `onError` callback, via a
   shared idempotent `cleanup()` closure (guards on `Set.remove`'s boolean
   return so a socket that hits both `.done` and `onError` only gets logged/
   removed once). Verified with a 20-connection concurrent stress test
   (`scripts` not checked in — see stress test pattern: open N sockets in
   threads, each sends a few commands, holds a few seconds, closes; poll
   the app's 流量 tab "連線數" before/after) — count returned to exactly 0
   after all clients disconnected. If "連線數" ever gets stuck non-zero
   again after clients have genuinely disconnected, suspect a new/different
   leak path, not a regression of this exact one.

## Related skills

`phone-termux-remote` (the phone's own SSH/adb connectivity, ZeroTier
recovery, Termux — check this first), `t25-xampp-dashboard` (T25's SSH
connection details/password if controlling T25 specifically — do not
hardcode T25's password here).
