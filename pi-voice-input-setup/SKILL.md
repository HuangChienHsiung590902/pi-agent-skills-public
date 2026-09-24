---
name: pi-voice-input-setup
description: "Maintain the local Pi voice input extension @mistgc/pi-voice-input on this Windows machine: install it into Pi's private npm workspace, fix F2/offline SenseVoice setup, force Traditional Chinese/Taiwan wording output, use the current/default microphone or AirPods-like headset preference, and troubleshoot ffmpeg DirectShow code -5 microphone errors. Use when the user mentions Pi voice input, F2 dictation, /voice, SenseVoice model, AirPods microphone switching, simplified-to-traditional output, or ffmpeg microphone errors in Pi."
compatibility: Windows host HCH, Pi coding agent, @mistgc/pi-voice-input 0.1.x, ffmpeg on PATH.
---

# Pi Voice Input Setup

This skill documents the working local setup for `@mistgc/pi-voice-input` in Pi.

## Important rule

Do **not** install this extension with global npm only.

Wrong / insufficient:

```powershell
npm install -g @mistgc/pi-voice-input
pi install @mistgc/pi-voice-input
```

Why:

- `pi install @mistgc/pi-voice-input` may treat it as a local path like `C:\Users\HCH\@mistgc\pi-voice-input`.
- Global npm installs under `C:\Users\HCH\AppData\Roaming\npm\node_modules`, but Pi loads packages from `~/.pi/agent/npm/` plus `~/.pi/agent/settings.json`.

Correct install location:

```bash
cd ~/.pi/agent/npm
npm install @mistgc/pi-voice-input opencc-js
```

Also ensure `~/.pi/agent/settings.json` contains:

```json
"packages": [
  "npm:omniroute-pi-ext-integration",
  "npm:pi-playwright",
  "npm:pi-mcp-adapter",
  "npm:@mistgc/pi-voice-input"
]
```

Then in Pi:

```text
/reload
```

## Current working configuration

Config file:

```text
~/.pi/agent/voice-input.json
```

Current intended values:

```json
{
  "hotkey": "f2",
  "mouseButton": "none",
  "modelPath": "~/.pi/agent/voice-input-models/sense-voice-small",
  "capture": {
    "tool": "auto",
    "ffmpegPath": "ffmpeg",
    "device": "default",
    "sampleRate": 16000,
    "channels": 1,
    "maxSeconds": 120
  },
  "output": {
    "caveatPrefix": "",
    "appendTrailingSpace": true,
    "chineseVariant": "taiwan"
  }
}
```

Meaning:

- `hotkey: "f2"` — press F2 to start/stop dictation.
- `mouseButton: "none"` — middle mouse trigger is intentionally disabled. It only works in Pi fullscreen TUI and is not worth the friction.
- `capture.device: "default"` — dynamically choose the current/default usable mic; prefer headset/Bluetooth/AirPods-like devices when present.
- `chineseVariant: "taiwan"` — convert SenseVoice simplified output to Traditional Chinese + Taiwan terms, e.g. `软件` -> `軟體`, `数据` -> `資料`, `网络` -> `網路`.

## Model

Model directory:

```text
~/.pi/agent/voice-input-models/sense-voice-small/
```

Expected files include:

```text
model.int8.onnx
tokens.txt
```

If missing, run in Pi:

```text
/voice model download
```

The downloaded `.tar.bz2` can be deleted after extraction to save space.

## Known local patches

The local install under:

```text
~/.pi/agent/npm/node_modules/@mistgc/pi-voice-input/extensions/voice-input/
```

has been patched beyond upstream. Backup files exist as `.bak`.

Patched files:

- `config.ts`
  - Adds `mouseButton?: "none" | "middle" | "right"`.
  - Adds `output.chineseVariant?: "none" | "traditional" | "taiwan"`.
- `formatter.ts`
  - Uses `opencc-js` to convert simplified Chinese to Traditional Chinese/Taiwan wording at output time.
  - Falls back to raw text if `opencc-js` fails, never drops dictated content.
- `audio-visualizer.ts`
  - Replaces the original scrolling block waveform with a cleaner smoothed pill meter: `🎙 REC ▐■■■··············▌`.
- `recorder.ts`
  - Fixes Windows DirectShow `device: "default"` path.
  - Fixes `parseDshowDeviceNames()` for ffmpeg builds that do **not** print a `DirectShow audio devices` header; scan lines containing `(audio)` instead.
  - Does **not** pass literal `audio=default` to ffmpeg; that causes `ffmpeg exited with code -5`.
  - Resolves the mic fresh on each recording so switching to AirPods/headset is picked up without reload.
  - If COM default-device query fails, ranks live DirectShow devices: prefer `airpod|bluetooth|headset|earbud|藍牙|耳機`, deprioritize webcam/virtual/stereo mix, otherwise first available mic.
- `index.ts`
  - Adds optional mouse-button toggle support, but config currently sets `mouseButton: "none"`.
  - `/voice status` displays `Mouse toggle`.

Important: `npm update` or reinstalling `@mistgc/pi-voice-input` may overwrite these patches. Reapply this skill's patch guidance if F2 works but Traditional Chinese/default mic behavior disappears.

## Troubleshooting

### After patching code, Pi still shows the old error

`/reload` may not always unload already-imported TypeScript extension modules. If a changed source file still appears not to take effect, fully exit and restart the Pi process. Example symptom: `recorder.ts` parser patch exists on disk, but Pi still says `No audio capture device found`.

Quick temporary workaround: set `capture.device` to the exact fixed DirectShow device name, reload, and later switch back to `default` after a full restart.

### F2 does nothing

Check:

1. Is the package installed in Pi's private npm workspace?

```bash
ls ~/.pi/agent/npm/node_modules/@mistgc/pi-voice-input
```

2. Is it listed in settings?

```bash
grep -n "pi-voice-input" ~/.pi/agent/settings.json
```

3. Reload or fully restart Pi:

```text
/reload
```

Then run in Pi:

```text
/voice status
```

If `/voice` command does not exist after reload, the extension was not loaded.

### Warning: speech model not found

This means the extension is loaded, but model files are missing. Run:

```text
/voice model download
```

### ffmpeg exited with code -5 / `Could not find audio only device with name [default]`

This is usually one of the upstream DirectShow bugs: either literal `audio=default` was passed to DirectShow, or the ffmpeg device parser returned an empty list because this ffmpeg build omits the `DirectShow audio devices` header. In the patched local version neither should happen.

If it returns after an update:

1. Restore/reapply the `recorder.ts` patch.
2. Or set a fixed device name from ffmpeg list:

```bash
ffmpeg -list_devices true -f dshow -i dummy
```

Example fixed device:

```json
"device": "麥克風排列 (適用於數位麥克風的 Intel® 智慧型音效技術)"
```

But preferred local behavior is still `"device": "default"` with the patched dynamic resolver.

### Need to test microphone outside Pi

```bash
ffmpeg -f dshow -i audio="麥克風排列 (適用於數位麥克風的 Intel® 智慧型音效技術)" -ar 16000 -ac 1 -t 3 -y /tmp/mictest.wav
```

Successful output produces a small WAV file.

### Want Traditional Chinese variants

Set in `~/.pi/agent/voice-input.json`:

```json
"chineseVariant": "taiwan"
```

Options:

- `taiwan` — Traditional + Taiwan terms; recommended.
- `traditional` — Traditional character conversion only.
- `none` — no conversion; keep model raw simplified output.

Then:

```text
/reload
```

### Mouse middle click trigger

The code supports:

```json
"mouseButton": "middle"
```

But it only works when Pi is launched with fullscreen TUI:

```powershell
pi --tui-mode fullscreen
```

In regular TUI mode, mouse reporting is not enabled; middle click never reaches the extension. Current preference is to keep:

```json
"mouseButton": "none"
```

and use F2.

## Quick status checklist

Run in Pi:

```text
/voice status
```

Healthy state should indicate:

- model ready/downloaded
- transcriber ready
- recorder/mic detected
- hotkey F2
- mouse toggle none

Then press F2, speak, press F2 again. Dictated Chinese should appear as Traditional Chinese/Taiwan wording.

---

## Conformance Addendum

## When to Use
Maintain the local Pi voice input extension @mistgc/pi-voice-input on this Windows machine: install it into Pi's private npm workspace, fix F2/offline SenseVoice setup, force Traditional Chinese/Taiwan wording output, use the current/default microphone or AirPods-like headset preference, and troubleshoot ffmpeg DirectShow code -5 microphone errors. Use when the user mentions Pi voice input, F2 dictation, /voice, SenseVoice model, AirPods microphone switching, simplified-to-traditional output, or ffmpeg microphone errors in Pi.

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
