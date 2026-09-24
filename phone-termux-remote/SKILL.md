---
name: phone-termux-remote
description: Remote-control and repair the user's Samsung phone (RFCW3049BHM) over USB ADB, SSH into Termux, and ADB over ZeroTier IP — get GPS location, run Termux/termux-api commands, diagnose sshd/service-daemon failures, repair stale authorized_keys, or install/log in Claude Code CLI on-device. Use when the user asks about "手機"/"phone" GPS location, connecting or reconnecting to the phone, USB ADB, Termux, SSH, sshd, ZeroTier, authorized_keys, or Claude Code on the phone.
triggers:
  - 手機 GPS
  - 手機座標
  - 連接手機
  - USB ADB
  - termux
  - 手機 ssh
  - sshd
  - ZeroTier
  - authorized_keys
  - 手機 claude code
  - 手機登入 claude
argument-hint: "[location|connect|repair|status]"
---

# phone-termux-remote Skill

## Purpose

The user's Samsung Galaxy A34 (serial `RFCW3049BHM`, model SM-A346) has a full remote-control
environment already set up: SSH key access into Termux, plus ADB over the same network. This
skill wraps that setup so you don't re-derive it (or re-fight its gotchas) every session.

**Always try `scripts/connect.sh` first** before doing anything else with this phone. It is
idempotent and fixes the two things that reset on every phone reboot (see Known Gotchas below).

## Connection Details

| What | Value |
|---|---|
| Phone | Samsung Galaxy A34 5G, serial `RFCW3049BHM` |
| ZeroTier IP | `10.145.119.96` (network `AI3`, ID `08752e18b15e0bce`) |
| SSH | `ssh -p 8022 -i ~/.ssh/id_ed25519 10.145.119.96` (user `u0_a846`, key-only, no password) |
| ADB over TCP | `adb -s 10.145.119.96:5555 ...` |
| Termux flavor | **GitHub** build (not Play Store — see Why below) |

## Scripts

Run these from this skill's `scripts/` directory (or with the full path). All are bash, run via
the Bash tool (Git Bash), not PowerShell.

- **`scripts/connect.sh`** — Run this FIRST, always. Checks SSH; if down, re-enables `adb tcpip` via USB
  and re-toggles the ZeroTier network switch, then retries SSH until it's up. No-ops if already
  connected. Needs the USB cable plugged in only if recovery is actually needed.
- **`scripts/get_location.sh [gps|network]`** — Prints live GPS JSON via `termux-location`. Defaults to
  `network` provider (near-instant, works indoors). Use `gps` outdoors for <10m accuracy (can take
  a while to get a satellite fix, `Ctrl+C`-able).
- **`scripts/adb_type.sh "<text>" [device_serial]`** — Only needed in the rare case SSH isn't set up yet
  and you must bootstrap by typing into Termux's on-screen terminal via `adb shell input text`.
  Handles the double-quoting gotcha (see Known Gotchas). Screenshot and confirm the line before
  sending Enter yourself.
- **`scripts/vibrate.sh [duration_ms]`** — Vibrates the phone via `termux-vibrate`. Verified working, no
  extra permission prompt needed. Default 1000ms.
- **`scripts/termux_api_menu.ps1`** — Interactive PowerShell text menu covering all 55 `termux-api`
  commands from the reference table below. Two-level lettered navigation (style loosely modeled
  on `D:\MAS_AIO.cmd`'s menu, per user request): top level shows the 12 categories as one
  single-page list keyed `A`-`L`; picking one drops into that category's items, keyed lowercase
  `a`-`j` (max category size is 10), also one page. `0` = back/exit, `R` = recheck phone
  connection, both work at either level. Every redraw calls `Clear-Host` + explicitly resets
  `CursorPosition` to `(0,0)` so the menu always redraws from the same fixed spot instead of
  drifting down the scrollback — this was a specific user complaint about the earlier flat
  55-item single-screen version, along with wanting it to fit on one page. After running an item
  it returns to that item's category submenu (not all the way to the top), so trying several
  commands in the same category doesn't require re-navigating. Prompts for arguments on commands
  that need them, requires y/n confirmation on risky ones (call, SMS send, wallpaper, wifi
  toggle, download), and auto-downloads+opens the output file for camera-photo /
  microphone-record. Run with `pwsh -File scripts/termux_api_menu.ps1` — must run in a real
  interactive terminal (not piped), since it uses `Read-Host` throughout. Calls `scripts/connect.sh` on
  startup. This is the tool to hand the user for "let me try these one by one" requests instead
  of running commands ad hoc over SSH. Camera-photo takes a throwaway warm-up shot, sleeps 2s,
  then takes the real shot — see Gotcha #8 for why.
- **`scripts/install_claude_code.sh [version]`** — Installs the Claude Code CLI into Termux, pinned to
  `2.1.112` by default (the last npm release with a pure-JS `cli.js` entry point — see "Claude
  Code CLI on Termux" below and Gotcha #10 for why anything newer is a dead end here). Also
  (re)creates a `cc` shortcut (`/data/data/com.termux/files/usr/bin/cc`, runs `claude
  --dangerously-skip-permissions` — the user's preferred short command) and sets
  `export DISABLE_AUTOUPDATER=1` in `.bashrc` so the CLI's own background auto-updater can't
  silently undo the version pin (Gotcha #14). Idempotent: safe to rerun, rewrites only the `alias
  claude=`/`cc`/`DISABLE_AUTOUPDATER` lines and leaves the rest of `.bashrc` intact.
- **`scripts/start_claude_login.sh [scratch_dir]`** — Starts `claude auth login` on the phone inside a
  detached tmux session (`claudelogin`), opens the OAuth URL in the phone's own Chrome via ADB,
  best-effort auto-taps the "Authorize" button (found by text via `uiautomator dump`, not
  hardcoded coordinates), switches the phone to landscape so a long returned code isn't cut off,
  and screenshots the result to `<scratch_dir>/claude_login_prompt.png` (default: this scripts/
  dir). Read the code out of that screenshot yourself, then hand it to
  `scripts/submit_claude_login_code.sh`. Requires USB ADB connected and Chrome already signed into the
  target claude.ai account (no in-browser login flow is automated, only the OAuth consent step).
- **`scripts/submit_claude_login_code.sh '<code>#<state>'`** — Sends the code into the waiting
  `claudelogin` tmux session, checks `claude auth status`, and cleans up (kills the tmux session,
  releases the wake-lock, restores auto-rotation). Prints SUCCESS/WARNING based on
  `"loggedIn": true`. Codes are single-use and short-lived — if this reports failure, rerun
  `scripts/start_claude_login.sh` for a fresh one rather than retrying the same code.

For anything else (running arbitrary Termux commands, `pkg install`, `termux-api` calls, file
ops), just SSH directly once `scripts/connect.sh` confirms it's up:
```bash
ssh -p 8022 -i ~/.ssh/id_ed25519 10.145.119.96 "<command>"
```

## SSH／Termux 遠端連線修復流程

當 `adb devices` 顯示手機為 `device`，但 SSH `10.145.119.96:8022` 連不上時，依下列順序處理。只在 SSH 已失效時使用 USB ADB 做修復；不要因 SSH 正常而重啟服務或覆寫金鑰。

1. **先執行既有恢復流程**：
   ```bash
   bash scripts/connect.sh
   ```
   若成功，直接以 SSH 執行 `echo SSH_OK` 驗證；若失敗，繼續用 USB serial `RFCW3049BHM` 診斷。

2. **確認 USB ADB、ADB TCP 與 ZeroTier**：
   ```bash
   adb devices -l
   adb -s RFCW3049BHM shell 'ip addr show tun0'
   adb -s RFCW3049BHM shell 'cat /proc/net/tcp /proc/net/tcp6'
   ```
   `tun0` 應包含 `10.145.119.96`。若 USB 裝置不是 `device`，先解鎖手機並在手機上接受 USB 偵錯授權；不要清除手機資料。

3. **檢查 Termux 服務監督程序**：
   ```bash
   adb -s RFCW3049BHM shell 'ps -A -o USER,PID,NAME,ARGS' | grep -Ei 'runsv(dir)?|svlogd|sshd'
   ```
   正常至少應看到 `runsvdir .../usr/var/service`、`runsv sshd`、`svlogd .../sshd` 與 `sshd -D -e`。只有 `com.termux` 或 `com.termux.api` 不代表 SSH 已啟動。

4. **若 `runsvdir`／`sshd` 不存在，從 Termux 使用者環境啟動 `service-daemon`**。Android ADB shell 的 `PATH`、`PREFIX`、`SVDIR` 可能不完整；必須明確設定，並用 `MSYS_NO_PATHCONV=1` 防止 Git Bash 改寫 Android 絕對路徑：
   ```bash
   MSYS_NO_PATHCONV=1 adb -s RFCW3049BHM shell 'run-as com.termux sh -c '\''
   export PREFIX=/data/data/com.termux/files/usr
   export SVDIR=$PREFIX/var/service
   export LOGDIR=$PREFIX/var/log
   export PATH=$PREFIX/bin:/system/bin
   $PREFIX/bin/service-daemon start
   '\'''
   sleep 3
   adb -s RFCW3049BHM shell 'ps -A -o USER,PID,NAME,ARGS' | grep -Ei 'runsv(dir)?|svlogd|sshd'
   ```
   若輸出顯示 `runsvdir .../usr/bin/runsvdir ...: not a directory`，表示先前的啟動命令被本機 shell 的變數展開或 ADB 引數重組弄壞；先只終止該次診斷確認的錯誤 `runsvdir` PID，再用上面的正確環境重試。不要用無差別 `pkill`，避免殺掉 SSH 或其他服務。

5. **確認 SSH port 與登入**：
   ```bash
   ssh -o ConnectTimeout=8 -o BatchMode=yes -p 8022 \\
     -i ~/.ssh/id_ed25519 10.145.119.96 \\
     'echo SSH_OK; id -un; hostname; printf "home=%s\\n" "$HOME"'
   ```
   預期使用者為 `u0_a846`，Home 為 `/data/data/com.termux/files/home`。

6. **若服務正常但出現 `Permission denied (publickey,...)`，檢查公鑰是否過期**：
   ```bash
   ssh-keygen -lf ~/.ssh/id_ed25519.pub
   MSYS_NO_PATHCONV=1 adb -s RFCW3049BHM shell \\
     'run-as com.termux cat /data/data/com.termux/files/home/.ssh/authorized_keys'
   ```
   比對本機 `~/.ssh/id_ed25519.pub` 與手機內容。只有確認本機私鑰對應的公鑰不在手機清單時，才追加公鑰：
   ```bash
   PUBKEY=$(cat ~/.ssh/id_ed25519.pub)
   MSYS_NO_PATHCONV=1 adb -s RFCW3049BHM shell \\
     "run-as com.termux sh -c \\\"grep -qxF '$PUBKEY' /data/data/com.termux/files/home/.ssh/authorized_keys || printf '%s\\\\n' '$PUBKEY' >> /data/data/com.termux/files/home/.ssh/authorized_keys\\\""
   ```
   優先追加而非覆寫 `authorized_keys`，保留手機上其他已授權的公鑰；確認後再重新執行 SSH 驗證。

## Termux:API Command Reference

Full list of `termux-api-package` scripts (55 commands, from the official
`termux/termux-api-package` repo `scripts/` dir). `termux-location` (Gotcha #4) and
`termux-telephony-deviceinfo` (Gotcha #7) are currently verified working on this phone, both
after granting a missing runtime permission via adb. Any other command may hit a similar
undocumented permission requirement the first time it's called via SSH — if it fails with a
`{"error": "Please grant the following permission..."}` JSON body (or fails silently), just run:
`adb -s RFCW3049BHM shell pm grant com.termux.api <permission_name>` (use the USB serial directly,
not the ZeroTier TCP address — see Gotcha #7). Check
`adb shell dumpsys package com.termux.api | grep permission` for the current grant state before
assuming the command itself is broken. Once a command is confirmed working and useful, promote it
to its own `scripts/*.sh` following the `scripts/get_location.sh` pattern rather than re-deriving the raw
command each time.

Invoke any of these over the existing SSH connection:
```bash
ssh -p 8022 -i ~/.ssh/id_ed25519 10.145.119.96 "termux-<command> [args]"
```

| Category | Commands |
|---|---|
| Device info | `termux-audio-info` ✅ verified, `termux-battery-status` ✅ verified, `termux-telephony-deviceinfo` ✅ verified, `termux-telephony-cellinfo` ✅ verified |
| Camera/media | `termux-camera-info` ✅ verified (no extra permission needed), `termux-camera-photo` ✅ verified (needs warm-up shot, see Gotcha #8), `termux-media-player`, `termux-media-scan` ✅ verified (bare invocation cleanly errors "missing file argument", not a permission issue — needs a real file path to actually scan), `termux-microphone-record` ✅ verified (needs `RECORD_AUDIO`, see Gotcha #7) |
| Comms | `termux-call-log` ✅ verified (needs `READ_CALL_LOG`, see Gotcha #7), `termux-telephony-call`, `termux-sms-list` ✅ verified (needs `READ_SMS`, see Gotcha #7), `termux-sms-inbox` ⚠️ deprecated — always prints "This script has been replaced by termux-sms-list", not a permission check, `termux-sms-send`, `termux-contact-list` ✅ verified (needs `READ_CONTACTS`, see Gotcha #7) |
| Location/sensors | `termux-location` ✅ verified, `termux-sensor` ✅ verified (`-l` to list) |
| UI/interaction | `termux-dialog` (untested — opens a blocking UI prompt on-device, don't fire-and-forget over SSH without expecting it to hang until answered), `termux-toast` ✅ verified, `termux-notification` ✅ verified, `termux-notification-list` ✅ verified (needs Notification access, see Gotcha #9), `termux-notification-remove`, `termux-notification-channel` ✅ verified (no `list` subcommand despite the name — only create/delete; bare invocation crashes with "unbound variable" under `set -u`, use `-h` or any 1-arg form to see usage safely), `termux-share` ✅ verified (bare invocation cleanly reports "Nothing to share"), `termux-wallpaper` |
| Output/control | `termux-vibrate` ✅ verified (`scripts/vibrate.sh`), `termux-torch` ✅ verified (on/off both work, no extra permission needed), `termux-brightness`, `termux-volume` ✅ verified, `termux-tts-speak` ✅ ran without error (exit 0, no output expected — audible result unverifiable remotely), `termux-tts-engines` ⚠️ returns empty (no TTS engine bound at call time — untested if a real `-speak` call first fixes this), `termux-speech-to-text` ⚠️ returns empty (exit 0) when fired over SSH with no one speaking — inconclusive, not confirmed broken |
| Clipboard | `termux-clipboard-get` ⚠️ always empty over SSH — Android 10+ only lets the *focused/foreground* app read the clipboard, and a background SSH-triggered call is never foreground. No permission fix exists for this (OS policy, not a grantable permission). `termux-clipboard-set` — untested whether the write itself lands (can't verify read-back for the same reason). |
| Network | `termux-wifi-connectioninfo` ✅ verified, `termux-wifi-scaninfo` ✅ verified, `termux-wifi-enable`, `termux-download` |
| Security | `termux-fingerprint` (untested — blocks waiting for a physical fingerprint scan, don't fire over SSH expecting a quick return), `termux-keystore` ✅ verified (bare invocation cleanly shows usage) |
| Special hardware | `termux-nfc` ✅ verified (bare invocation shows usage), `termux-usb` ✅ verified (`-l` → `[]`, no USB device attached), `termux-infrared-frequencies` ✅ ran without error — returns empty because this phone (Galaxy A34) has no IR blaster hardware, not a bug, `termux-infrared-transmit` (same hardware caveat, untested) |
| SAF file access | `termux-saf-ls` ✅ verified (bare invocation shows a clean argument-count error, not a hang), `termux-saf-dirs` ✅ verified (`[]` — no folder granted yet via `termux-saf-managedir`), `termux-saf-stat`, `termux-saf-create`, `termux-saf-mkdir`, `termux-saf-write`, `termux-saf-read`, `termux-saf-rm`, `termux-saf-managedir`, `termux-storage-get` |
| Scheduling | `termux-job-scheduler` ✅ verified (bare invocation shows a clean "No script path given" message) |
| Service control | `termux-api-start`, `termux-api-stop` (internal, not a data command) |

## Claude Code CLI on Termux

Installed via `scripts/install_claude_code.sh`, pinned to npm `@anthropic-ai/claude-code@2.1.112`. Signed
in as `hch590902@gmail.com` (Pro subscription) via `scripts/start_claude_login.sh` +
`scripts/submit_claude_login_code.sh`, verified end-to-end with a real `claude --print` call. Run it with
`ssh -p 8022 -i ~/.ssh/id_ed25519 10.145.119.96` then `claude`, or `cc` for
`claude --dangerously-skip-permissions` (user's preferred shortcut, both set up by
`scripts/install_claude_code.sh`). See Gotcha #10 before ever bumping the version, #11-#13 before touching
the login flow again, and #14 if `claude`/`cc` suddenly break with "native binary not installed"
or "Cannot find module .../cli.js" after previously working.

## Known Gotchas (why the scripts exist)

1. **ADB TCP mode resets to USB-only on every phone reboot.** `adb tcpip 5555` must be re-run
   over USB after each reboot before `adb -s <ip>:5555` will work again. `scripts/connect.sh` handles this.
2. **ZeroTier's per-network toggle does not persist across reboot** — it comes back up as OFF
   even though the app itself auto-starts. Fastest fix found: tap the toggle at screen coords
   `(1004, 384)` in `com.zerotier.one/.ui.NetworkListActivity`. A *single* tap — double-tapping
   cancels itself back off. `scripts/connect.sh` handles this.
3. **sshd auto-start on boot needs supervision, not a bare `sshd` call.** The Termux:Boot script
   (`~/.termux/boot/start-sshd.sh` on the phone) must call `termux-wake-lock` +
   `service-daemon start` (runit-supervised), not just run `sshd` directly — a bare `sshd`
   process gets reaped by Android within ~1-2 minutes of boot since it has no foreground-service
   protection. Already fixed and verified to survive 7+ minutes uninterrupted. If sshd ever stops
   auto-starting again, re-check this file's content on the phone.
4. **`termux-location` returns silently empty (exit 0, no output) when run via SSH/background**
   unless `com.termux.api` also holds `ACCESS_BACKGROUND_LOCATION` (not just FINE/COARSE).
   Foreground-typed commands worked without it; pure background SSH calls didn't. Already granted
   via `adb shell pm grant com.termux.api android.permission.ACCESS_BACKGROUND_LOCATION` — if a
   fresh Termux:API reinstall ever happens, redo this grant.
5. **`adb shell input text "multi word string"` silently mangles/truncates** unless double-quoted:
   the local shell's quotes don't survive the trip through `adb shell`, which rejoins argv with
   spaces before sending to the remote shell. Correct form:
   `adb shell "input text '<the text>'"` (single quotes on the *remote* side). `scripts/adb_type.sh`
   already does this — use it instead of raw `adb shell input text`.
6. **Termux (Play Store build) and Termux:API/Termux:Boot (GitHub builds) refuse to talk to each
   other** — Termux:API isn't on Play Store yet (known upstream issue termux-play-store/termux-apps#29),
   and the Play-flavored main Termux app hard-blocks non-Play companion apps. All three
   (Termux, Termux:API, Termux:Boot) are now installed from GitHub releases to match. If any of
   them ever gets reinstalled from Play Store by accident, this breaks again — reinstall the
   GitHub `.apk` from the matching `termux/<repo>` GitHub releases page instead.
7. **Each `termux-api` command needing a "dangerous" Android permission fails on first use** with
   a JSON body like `{"error": "Please grant the following permission to use this command:
   android.permission.X"}` until `com.termux.api` holds that permission. Confirmed so far:
   `termux-telephony-deviceinfo` → `READ_PHONE_STATE`; `termux-camera-photo` → `CAMERA`;
   `termux-contact-list` → `READ_CONTACTS`; `termux-microphone-record` → `RECORD_AUDIO`;
   `termux-call-log` → `READ_CALL_LOG`; `termux-sms-list` → `READ_SMS`. Fix is
   always the same: `adb -s RFCW3049BHM shell pm grant com.termux.api android.permission.<X>` —
   run this against the **USB serial** (`RFCW3049BHM`), not the ZeroTier TCP address
   (`10.145.119.96:5555`), since ADB TCP mode is frequently down (Gotcha #1 — SSH being up
   doesn't imply `adb tcpip` survived the last reboot) and USB is always available for this kind
   of fix. Likely applies to any not-yet-tried command in the reference table above that touches
   telephony/SMS/contacts/mic/storage — same fix pattern, just swap the permission name from the
   error message. If a fresh Termux:API reinstall ever happens, redo all previously-granted
   permissions.
8. **`termux-camera-photo` produces visibly out-of-focus JPEGs on a single cold shot.** Root
   cause confirmed by reading the installed app's source
   (`CameraPhotoAPI.java`, `termux/termux-api` master): it sets
   `CONTROL_AF_MODE_CONTINUOUS_PICTURE` but never waits for `CONTROL_AF_STATE ==
   FOCUSED_LOCKED` — it just runs a hardcoded `Thread.sleep(500)` preview then captures
   regardless of whether focus actually converged. No CLI flag controls this (the wrapper script
   only takes `-c camera-id`); a PR adding manual focus/`preview_time` control
   (termux/termux-api#694) exists but is **not merged** (maintainers are holding feature PRs).
   Verified workaround: take one throwaway shot, `sleep 2`, then take the real shot — the real
   shot comes out sharp because the sensor/AF has settled from the first open. Confirmed via
   direct side-by-side test (single cold shot = blurry text, warm-up+delay shot = sharp text on
   the same physical subject). Already baked into `scripts/termux_api_menu.ps1` item 6; apply the same
   two-shot pattern to any raw `termux-camera-photo` call run manually over SSH.
9. **`termux-notification-list` hangs forever (no output, no error) instead of failing fast.**
   Unlike Gotcha #7's permission errors, this command needs **Notification access**
   (`BIND_NOTIFICATION_LISTENER_SERVICE`) — a special system-level access toggle (Settings >
   Notification access), not a normal runtime permission, so `pm grant` does nothing for it. Root
   cause confirmed by reading `NotificationListAPI.java`: without this access,
   `NotificationService.get()` returns `null` and the subsequent call throws inside the app with
   no reply ever sent back to the waiting shell command — hence the hang instead of a clean JSON
   error. Fix: `adb -s RFCW3049BHM shell cmd notification allow_listener
   'com.termux.api/com.termux.api.apis.NotificationListAPI$NotificationService'` (verify with
   `adb shell settings get secure enabled_notification_listeners | tr ':' '\n' | grep termux`).
   **Gotcha: the `$NotificationService` inner-class name gets silently eaten if not escaped
   right** — same double-shell-parsing trap as Gotcha #5. Wrap the whole `cmd notification ...`
   invocation in **double quotes** (not single) for the outer `adb shell "..."` call, and escape
   the inner `$` as `\$` so local bash doesn't expand it either — otherwise `adb shell` silently
   registers the truncated component `com.termux.api/com.termux.api.apis.NotificationListAPI`
   (no `$NotificationService` suffix), which does nothing and still hangs. If a fresh Termux:API
   reinstall happens, redo this grant (and note it survives `pm grant` resets differently since
   it's stored in `enabled_notification_listeners`, not the app's permission list).

10. **`@anthropic-ai/claude-code` 2.1.113+ ships a native binary with no Android build, making it
    a dead install on Termux.** Confirmed by reading the installed package's own
    `cli-wrapper.cjs`/`install.cjs`: newer versions fetch a platform binary via
    `optionalDependencies` (`bin/claude.exe`, despite the name — it's the actual CLI, not a
    Windows PE), and the published `optionalDependencies` list for 2.1.218 has zero
    `linux-*-android` entries even though the wrapper's own `PLATFORMS` map already has
    placeholder logic for `linux-arm64-android`. Running it prints "claude native binary not
    installed" and exits 1; manually running `install.cjs` confirms with "Native binaries for
    linux-arm64-android are not available on this release channel." **2.1.112 is the last
    version published with the old pure-JS `bin: cli.js` entry point** (verified via `npm view
    @anthropic-ai/claude-code versions` + inspecting each version's `bin`/`optionalDependencies`
    fields from the registry) — it runs under plain Node regardless of CPU/OS, so it works fine
    here. `scripts/install_claude_code.sh` pins to this. If Anthropic ever publishes an
    `linux-arm64-android` binary, a newer version might start working again, but don't assume it
    without checking `optionalDependencies` for that package first.
11. **`pkill -f '<pattern>'` run over SSH can kill the SSH session's own remote shell if the
    pattern also appears in the invoking command line** — the wrapper shell running your one-line
    SSH command has that entire line as its own `cmdline`, and `pkill -f` scans ALL processes'
    cmdlines, including it. Symptom: the SSH call itself exits 255 with zero output, even though
    the target process would otherwise have matched fine. Fix: the classic bracket trick — e.g.
    `pkill -f '[c]li.js auth login'` instead of `pkill -f 'cli.js auth login'`. The bracketed
    pattern still matches the plain target process (`[c]` in a regex is just a character class
    containing `c`), but does NOT match your own invoking command line, since that literally
    contains the bracket characters (`[c]li.js...`) which breaks the contiguous-match the regex
    needs against itself. `scripts/start_claude_login.sh` uses this for its cleanup step.
12. **A URL containing literal `&` gets truncated by `adb shell` unless the whole URL is one
    single-quoted argument on the *remote* shell's side.** `adb shell am start ... -d "$URL"` with
    only local/outer quoting still arrives at the phone as an unquoted string containing `&`,
    which the remote shell interprets as its own background-job operator — the URL gets cut at
    the first `&`, and pages depending on later query params (e.g. `client_id`) fail with "Missing
    client_id parameter". Correct form: `adb shell "am start ... -d '$URL'"` — single quotes
    around the URL survive because they're INSIDE the one big double-quoted string handed to `adb
    shell`, so the remote shell sees them as its own quoting, not as literal characters.
    `scripts/start_claude_login.sh` does this.
13. **The Claude Code OAuth authorization code can be wider than the phone's portrait screen, and
    Chrome doesn't wrap it — the tail gets silently cut off in a screenshot.** Rotating to
    landscape (`settings put system user_rotation 1` after disabling
    `accelerometer_rotation`) roughly doubles visible width and was enough to show the whole code
    in practice; if a future code is even longer, zooming out the page or scrolling the text field
    horizontally are the fallbacks. Separately, the returned code has the shape `<code>#<state>`,
    and the `state` half is an exact echo of the `state=` query param from the original OAuth URL
    — cross-checking the two catches OCR/eyeball misreads that a font can hide (this caught a
    `0`-vs-`O` misread live: `SGYUf0Us` in the URL's `state=` vs. `SGYUfOUs` as first transcribed
    from the screenshot). `scripts/start_claude_login.sh` prints both so the check is easy; always do it
    before calling `scripts/submit_claude_login_code.sh`, since a wrong character means a wasted
    single-use code and a full restart of the flow.

14. **Claude Code's built-in background auto-updater silently reinstalls the latest npm version on
    a later launch, undoing the 2.1.112 version pin from Gotcha #10.** Observed live: right after
    a clean pinned install, running the CLI once (via a user-made `cc` shortcut) was enough for
    something in the background to bump the global npm package to `2.1.218` within about a
    minute — no explicit `npm install`/`claude update` was run. Since 2.1.218+ has no `cli.js` at
    all (only `bin/claude.exe` + `cli-wrapper.cjs`, the native-binary architecture from Gotcha
    #10), both the `claude` alias and any hardcoded-path shortcut like `cc` broke immediately
    afterward — `cc` failed with a Node `MODULE_NOT_FOUND` on `cli.js` (file gone), while `claude`
    (same path, checked moments apart) surfaced the `bin/claude.exe` shim's "claude native binary
    not installed" text, consistent with an install that was mid-flight at that exact moment.
    Fix: reinstall the pinned version (`scripts/install_claude_code.sh`) and set
    `export DISABLE_AUTOUPDATER=1` in `.bashrc` — `scripts/install_claude_code.sh` now does the latter
    automatically. If this env var ever stops being honored (future CLI versions sometimes rename
    these), grep the current `cli.js` for `AUTOUPDATER` to find the actual variable name before
    assuming the fix stopped working.

15. **SSH public-key auth can fail with `Permission denied (publickey,...)` even though sshd is
    running fine and ZeroTier/tun0 is up** — if the local `~/.ssh/id_ed25519` keypair was
    regenerated at some point (e.g. after a fresh setup), the phone's
    `~/.ssh/authorized_keys` can still hold the *old* public key, so the new private key no
    longer matches anything the phone trusts. `scripts/connect.sh` doesn't detect or fix this — it only
    handles ADB-tcpip/ZeroTier-toggle resets (Gotchas #1-2), not a stale `authorized_keys`.
    Diagnose: `adb -s RFCW3049BHM shell run-as com.termux cat
    /data/data/com.termux/files/home/.ssh/authorized_keys` (direct `adb shell cat` on that path
    gets `Permission denied` — must go through `run-as com.termux`) and diff it against
    `cat ~/.ssh/id_ed25519.pub` locally. Fix (USB only, since SSH itself is what's broken):
    ```bash
    PUBKEY=$(cat ~/.ssh/id_ed25519.pub)
    adb -s RFCW3049BHM shell "run-as com.termux sh -c \"echo '$PUBKEY' > /data/data/com.termux/files/home/.ssh/authorized_keys\""
    ```
    Note: in Git Bash, prefix any `adb shell run-as ... /data/...` command with
    `MSYS_NO_PATHCONV=1` when just *reading* a path directly (`adb -s ID shell run-as com.termux
    cat /data/...`), otherwise MSYS rewrites the leading `/data/...` into a bogus Windows path —
    same class of gotcha as #5/#12's quoting traps. The `echo ... > path` form above avoids the
    problem because the redirect target is inside the double-quoted remote command string, not a
    bare leading-slash argument.

16. **On a fresh Windows machine (or after a driver reinstall), `adb devices` shows nothing over
    USB even with the cable in and USB debugging authorized** — Device Manager (under "通用序列匯
    流排裝置"/"其他裝置") shows an `ADB Interface` entry with a yellow warning triangle,
    `DEVPKEY_Device_ProblemCode` = **18** (`CM_PROB_REINSTALL` — "reinstall the drivers for this
    device"). The specific broken driver package on this Samsung phone is `ssudadb.inf`
    (`SAMSUNG Electronics`, class `AndroidUsbDeviceClass`) — find its `oemNN.inf` alias with
    `pnputil /enum-drivers` (search for `ssudadb.inf` in the `Original Name` field). Fix needs an
    **elevated** PowerShell (ask the user to run it, don't self-elevate):
    ```powershell
    pnputil /delete-driver oemNN.inf /uninstall /force
    pnputil /scan-devices
    ```
    then physically unplug/replug the USB cable so Windows re-enumerates and reinstalls a clean
    copy. If Device Manager afterward shows the device sitting under "其他裝置" with a plain grey
    question mark (not the yellow triangle) instead of resolving automatically, right-click →
    Update driver → Search automatically; if THAT reports "Windows 為您的裝置安裝驅動程式時發生
    錯誤 / 此裝置的其中一個安裝程式目前無法執行安裝" even though it found the correct
    `SAMSUNG Android ADB Interface` driver by name, that's a stuck Windows driver-install
    subsystem — a full reboot resolves it essentially every time; there isn't a reliable
    non-reboot fix worth chasing first. Also re-check the phone side once ADB reconnects: Android
    will show a fresh "Allow USB debugging?" RSA-fingerprint dialog on this new pairing — approve
    it (tick "always allow") before `adb devices` will report `device` instead of `unauthorized`.
17. **ADB UI automation on this phone (`input tap`, `uiautomator dump`) is meaningfully less
    reliable over the ZeroTier-TCP transport (`adb -s 10.145.119.96:5555`) than over USB
    (`adb -s RFCW3049BHM`)** — taps that land correctly over USB get intermittently misread as
    swipes/long-presses over TCP (visible as the screen sliding sideways mid-gesture, or a
    long-press context menu popping up instead of a click), almost certainly latency-related
    (`adb pull` throughput was ~1MB/s over TCP vs 20-35MB/s over USB in side-by-side tests on this
    connection). **Prefer USB for any UI-automation-heavy task**; TCP is fine for simple
    fire-and-forget commands (`termux-*`, `screencap`) where timing doesn't matter.

    - `uiautomator dump` can return a **stale/mixed** accessibility tree during app transitions —
      seen returning nodes from two different Activities simultaneously (e.g. a chat app's title
      bar overlapping the previous screen's bottom nav bar) even when a fresh `screencap` taken a
      moment later shows a perfectly normal single screen. Don't trust a dump that looks like two
      screens spliced together — re-dump after an extra second, or just fall back to
      `screencap` + eyeballing pixel coordinates.
    - Any `adb shell <cmd> /sdcard/...` argument starting with `/` gets silently mangled by MSYS
      path conversion in Git Bash **unless** you set `MSYS_NO_PATHCONV=1` for that command — this
      applies to `uiautomator dump /sdcard/ui.xml` just as much as it does to `pull`/`push`
      (Gotcha #5/#12/#15's family of bugs). A mangled dump path silently succeeds but writes
      nowhere useful, and `pull` of the real `/sdcard/ui.xml` then just re-fetches yesterday's
      file — easy to mistake for "the UI hasn't changed" when actually the write never happened.
    - When an app's back-stack gets exhausted mid-automation (e.g. repeated `keyevent 4`/back
      presses), Android can fall through to whatever app was previously in the foreground instead
      of exiting cleanly — don't assume back-button presses stay within the app you think you're
      driving. `am force-stop <package>` + `am start -n <package>/<activity>` gives a guaranteed
      clean single-app state; use that instead of chained back-presses when the current screen
      looks wrong or unexpected.
    - List-based UIs that live-update from a background source (e.g. a chat app whose topic list
      reorders as new messages arrive) can race a screenshot-then-tap sequence — the row you
      screenshotted may no longer be at that y-coordinate by the time the tap fires. Minimize the
      screenshot→tap delay, or navigate by an identity-based method (search-by-name, a direct deep
      link) instead of a remembered row position when the list is known to be live.
    - Multi-select modes in apps like Telegram can trigger from what looks like a normal single
      tap (e.g. tapping a chat row when the touch registers slightly off or as a fast
      double-event) — if a screenshot shows selection checkmarks / a count in the header where
      there shouldn't be one, cancel immediately (usually an "X" at top-left) before tapping
      anything that could be a bulk-delete action.

18. **從 Android ADB shell 直接啟動 Termux `service-daemon` 可能失敗**：ADB 的環境通常沒有
    Termux 的 `PREFIX`、`PATH` 與 `SVDIR`，可能看到 `mkdir: '/var': Read-only file system`、
    `start-stop-daemon: not found`，或因引數／變數被錯誤重組而出現 `runsvdir ...: not a directory`。
    必須透過 `run-as com.termux`，明確設定 `PREFIX=/data/data/com.termux/files/usr`、
    `SVDIR=$PREFIX/var/service`、`LOGDIR=$PREFIX/var/log` 與 `PATH=$PREFIX/bin:/system/bin`，
    再執行 `$PREFIX/bin/service-daemon start`。若錯誤的 `runsvdir` 已在背景執行，先依 PID
    精準終止它再重試；不要用無差別 `pkill -f`。

19. **本次修復實測結果**：USB ADB 裝置 `RFCW3049BHM`（Samsung `SM-A3460`，Android 16）
    正常；`tun0` 有 `10.145.119.96`；以正確 Termux 環境啟動後可看到 `runsvdir`、`runsv sshd`、
    `svlogd` 與 `sshd -D -e`；SSH `10.145.119.96:8022` 可成功以 `u0_a846` 登入。當出現
    SSH 公鑰拒絕時，本機 `~/.ssh/id_ed25519.pub` 與手機 `authorized_keys` 曾不一致；追加
    對應公鑰後已通過 `echo SSH_OK` 驗證。

## Verifying sshd survives a reboot (only if you suspect it broke again)

```bash
adb -s RFCW3049BHM reboot
# wait ~90s+ for boot, then WITHOUT opening the Termux app manually:
adb -s RFCW3049BHM shell pgrep -fla sshd
# expect: "runsv sshd", "svlogd ...", "sshd -D -e" — all three, unsupervised bare "sshd" alone means it will die soon
```

---

## Conformance Addendum

## When to Use
Remote-control the user's Samsung phone (RFCW3049BHM) over SSH into Termux and/or ADB over ZeroTier IP — get GPS location, run Termux/termux-api commands, check status, install/log in Claude Code CLI on-device. Use when the user asks about "手機"/"phone" GPS location, running commands on the phone's Termux, reconnecting to the phone remotely, or installing/logging into Claude Code on the phone.

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
