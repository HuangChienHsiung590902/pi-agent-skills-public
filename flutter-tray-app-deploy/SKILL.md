---
name: flutter-tray-app-deploy
description: Build, package, deploy, and troubleshoot flutter_tray_app (product name "AssetAgent" / 資產管理代理程式) — a Flutter Windows desktop tray app that reports this machine's hardware specs to aipower and accepts remote shell commands over a Jocket long connection. Use this skill whenever the user mentions flutter_tray_app, AssetAgent, 資產管理 tray/agent app, T25, GRAM asset status, or asks to rebuild/repackage/reinstall/redeploy this app on this machine or a remote one (e.g. T25 at 10.145.119.209), or reports it showing offline/連不上/離線 in the aipower asset list, or a flutter_windows.dll crash tied to this specific app.
---

# flutter_tray_app (AssetAgent) — build, deploy, troubleshoot

## Where things live

- **Source**: `D:\Github\flutter_tray_app` — this is the ONLY real copy. It used to
  live at `C:\Users\HCH\flutter_tray_app` before being reorganized under
  `D:\Github`; that old path is dead. If you ever find a near-empty
  `C:\Users\HCH\flutter_tray_app` folder (just a `windows\flutter\ephemeral`
  remnant) lying around, that's leftover from something running with a stale
  path — not a sign of lost work, and not a place to develop from.
- **Backend**: the `aipower-app` Docker container on this same machine (see the
  `aipower-docker-local` skill), reached at `https://hch.james-huang.org` over
  the Cloudflare Tunnel (see `cloudflare-tunnel` skill) — the app never talks to
  `localhost`, so the exact same build works on any machine it's installed on.
  There is no more "Lab2" native install; if anything you find references Lab2
  for this backend, it's stale.
- **Installed locations**: `%LOCALAPPDATA%\Programs\AssetAgent\flutter_tray_app.exe`
  under whichever Windows account runs it (per-user install, not machine-wide —
  see "Remote deploy" below for why that matters). Credentials live alongside at
  `%APPDATA%\flutter_tray_app\{agent_credentials.bin,user_login.bin}`, DPAPI-encrypted
  to that specific Windows account.
- **Known machines**: this machine ("GRAM"), and T25 at `10.145.119.209` (the app
  itself runs under the `James` account there, a different session from whatever
  admin account you SSH in as). **SSH connection details are stale here** — the
  key-based/port-22 setup this used to describe stopped working; as of
  2026-07-28 T25 only accepts SSH on **port 2222**, password auth only (no
  working key), and `sshpass` fails silently on this machine's Git Bash (use
  `plink` instead). Full current connection method + why is documented in the
  `t25-xampp-dashboard` skill — check there before assuming port 22 works.

## Auth model (password-only — API Token mode was removed)

`AgentCredentials` is just `{loginName, password}`. Both background services
log in fresh with this password every cycle:
- `AutoReportService` — every 30 min (or every 1 min while retrying after a
  failure), logs in via `Qs.OnlineUser.login.data` and POSTs the asset spec.
- `AgentControlService` — maintains a persistent Jocket (WebSocket-based)
  connection for remote command execution. The Jocket handshake only accepts a
  `tokenId` (no username/password param), so this service silently exchanges
  the stored password for a fresh `tokenId` via `AssetLocalApiClient.applyToken`
  every time it needs to (re)connect — this is an internal protocol detail, not
  a second "auth mode" choice for the user.

**Why there's no more API Token mode**: an earlier version let the user choose
a persistent "API Token" instead of storing a password — but that token was
bound to the aipower server's in-memory session store, so *any* backend
restart (e.g. `docker restart aipower-app`) silently invalidated every
outstanding token, and the app had no password on hand to get a new one
automatically. Password mode sidesteps this entirely: every cycle re-derives
everything fresh from the stored password, so a backend restart just means
the *next* cycle logs in again like nothing happened. If you're ever tempted
to bring back a token/session-persistence option for this app, know that this
is the exact failure mode it re-introduces.

If you see a "設定自動回報帳號" dialog pop up unexpectedly after an upgrade, it's
because the previously-stored credentials file was in the old token format
(no password saved) and no longer parses — this is expected, one-time, and
needs the user to type their password in once. It cannot be filled in via SSH
or scripting; the user has to be physically at that machine's screen.

## Build & package a new installer

The project's own `installer/build-installer.sh` already does this (rebuild +
Inno Setup packaging in one step) — don't hand-roll the ISCC/flutter commands
again, just call it:

```bash
# from anywhere; SRC_DIR inside the script points at D:\Github\flutter_tray_app
D:/Github/flutter_tray_app/installer/build-installer.sh          # keep current version
D:/Github/flutter_tray_app/installer/build-installer.sh 1.0.7    # bump version first
```

Output: `D:\Github\flutter_tray_app\installer\Output\AssetAgentSetup-<version>.exe`.

Bump the version (second form) whenever the change is behaviorally meaningful
(new feature, bug fix affecting runtime behavior) — not for pure formatting/lint
cleanup. If `SRC_DIR` in that script ever stops matching reality (project moved
again), fix it there before anything else; a stale `SRC_DIR` silently builds
from the wrong/empty directory instead of failing loudly.

**Sanity check after building**: `build\windows\x64\runner\Release\flutter_tray_app.exe`
(the native runner shell) keeps almost the same file size/timestamp across
rebuilds even when the Dart code changed — that's normal, not a sign the build
didn't pick up your changes. The actual compiled Dart logic is
`build\windows\x64\runner\Release\data\app.so`; check *that* file's timestamp
if you need to confirm a rebuild really happened.

Always run `flutter analyze lib/` after editing source and before packaging —
it's fast and catches the obvious stuff before you burn time on a full build.

## Deploy

Use the bundled scripts — don't re-derive the install/launch/verify sequence
by hand each time, it has sharp edges (see Gotchas below) that are easy to
re-discover the hard way.

**This machine (GRAM)**:
```powershell
pwsh -File "C:\Users\HCH\.claude\skills\flutter-tray-app-deploy\scripts\scripts/deploy-local.ps1" -InstallerPath "D:\Github\flutter_tray_app\installer\Output\AssetAgentSetup-<version>.exe"
```
Kills any running/zombie instance, silent-installs, launches, and verifies
you're left with exactly one connected instance.

**A remote machine (e.g. T25)**:
```bash
bash "C:\Users\HCH\.claude\skills\flutter-tray-app-deploy\scripts\scripts/deploy-remote.sh" <ssh-host> <ssh-admin-user> <target-windows-user> <local-installer-path>
# e.g.
bash "C:\Users\HCH\.claude\skills\flutter-tray-app-deploy\scripts\scripts/deploy-remote.sh" 10.145.119.209 administrator James "D:/Github/flutter_tray_app/installer/Output/AssetAgentSetup-1.0.6.exe"
```
This exists because the app installs *per-user* (`PrivilegesRequired=lowest`,
`DefaultDirName={autopf}` resolves to that user's own Program Files
equivalent) — SSH-ing in as an admin account and running the installer
directly would install it into the *admin's* profile, not the target user's.
The script routes both the install and the launch through
`schtasks /ru <user> /it` so they run as if that user double-clicked it
themselves.

Both scripts print what's happening at each step; read the output rather than
re-running blind if something looks off.

## Gotchas (all previously hit, don't rediscover the hard way)

- **Never run two instances at once.** They fight over the same remote-control
  connection slot (same `assetCode`), and the aipower asset list will show the
  machine as offline even though *a* copy of the app is clearly running. Before
  trusting an "offline" report, check `Get-Process flutter_tray_app` shows
  exactly one PID.
- **Never externally manipulate this app's window** — no `ShowWindow`,
  `SetForegroundWindow`, `IsIconic`, etc. from outside the app (e.g. via
  P/Invoke to take a "confirmed foreground" screenshot). This reliably crashes
  `flutter_windows.dll` (same offset every time: `0x1cda0`, exception
  `0xc000041d`/`0xc0000005`) because the `window_manager` plugin is already
  managing this window's state internally and fights with any outside caller
  doing the same. If you need to see what's on screen, ask the user to look
  themselves, or use a purely passive `CopyFromScreen` on a window that's
  already visible — don't force it to the foreground first.
- **Indirect launches (scheduled task in another session, RDP, etc.) may crash
  once or twice before stabilizing.** This is a known Flutter-engine-init quirk
  specific to how this app gets launched non-interactively, not a new bug.
  Wait ~15-30s and check the PID is still the same one before concluding it
  failed; if it crashed, just launch it again.
- **Installer exit code 5** = Inno Setup's RestartManager couldn't close an app
  still holding the target files, and defaulted to Abort (silent mode suppresses
  the Abort/Retry/Ignore box). The cause is always some copy of
  `flutter_tray_app.exe` still alive — including one `Get-Process` can miss
  mid-teardown; check `Get-CimInstance Win32_Process -Filter "Name='flutter_tray_app.exe'"`
  too, kill it, and retry. Both deploy scripts already do this defensively.
- **Bash's `start ""` does not reliably detach a launched GUI process** in this
  environment — it can end up waiting on the long-running app and hang for the
  tool's full timeout. Prefer PowerShell `Start-Process` (no `-Wait` for the
  *launch* step; use `-Wait` only for the *installer*, so you get a real exit
  code) or bare `"$EXE" &` + `disown` in bash.
- **`AssetLocalApiClient.applyToken` is still legitimate** — don't confuse it
  with the removed "API Token" *user-facing* auth mode. It's the internal
  mechanism `AgentControlService` uses to get a `tokenId` for the Jocket
  handshake from the stored password; it's not persisted anywhere and gets
  re-derived on every reconnect.

---

## Conformance Addendum

## When to Use
Build, package, deploy, and troubleshoot flutter_tray_app (product name "AssetAgent" / 資產管理代理程式) — a Flutter Windows desktop tray app that reports this machine's hardware specs to aipower and accepts remote shell commands over a Jocket long connection. Use this skill whenever the user mentions flutter_tray_app, AssetAgent, 資產管理 tray/agent app, T25, GRAM asset status, or asks to rebuild/repackage/reinstall/redeploy this app on this machine or a remote one (e.g. T25 at 10.145.119.209), or reports it showing offline/連不上/離線 in the aipower asset list, or a flutter_windows.dll crash tied to this specific app.

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
