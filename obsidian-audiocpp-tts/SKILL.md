---
name: obsidian-audiocpp-tts
description: Manage the "audiocpp TTS Reader" Obsidian plugin (D:\OB\.obsidian\plugins\audiocpp-tts\) that reads the active note or selection aloud via the self-hosted audiocpp-server TTS engine on 10.145.119.19:7861. Use when the user wants to edit/redeploy this plugin, add or change a TTS voice/character, debug inconsistent voice across a reading, understand why a settings-page value doesn't match what's saved, or otherwise work on Obsidian text-to-speech.
---

# obsidian-audiocpp-tts: Obsidian TTS reader plugin

## What this is

A hand-written, no-build-step Obsidian community plugin at `D:\OB\.obsidian\plugins\audiocpp-tts\`:
- `manifest.json` — id `audiocpp-tts`, must also be listed in `D:\OB\.obsidian\community-plugins.json` to auto-enable.
- `main.js` — plain CommonJS (`require("obsidian")`), no TypeScript/esbuild pipeline. Edit directly.
- `data.json` — Obsidian-managed settings storage (per-vault), NOT source-controlled, holds the user's live config (server URL, voice presets, active preset, seed, temperature, chunk size).

It calls `audiocpp-server` (see skill `whisper-service`'s sibling container, and `audiocpp-realtime-web` for the browser-facing proxy version of the same backend) **directly** at `http://10.145.119.19:7861/v1/audio/speech` — not through the `audiocpp-web` proxy on port 8861. Obsidian's `requestUrl` API is used instead of `fetch` specifically because it bypasses CORS/Electron restrictions.

## Architecture (main.js)

- `markdownToPlainText(md)` — strips frontmatter, code blocks, images, wikilinks `[[..]]`, links, headings, callouts, lists, emphasis, tables, HTML tags before sending text to TTS.
- `splitIntoChunks(text, maxLen)` — splits on sentence-ending punctuation (`。！？.!?\n`), packing up to `chunkSize` chars per chunk (hard-splits only if a single sentence exceeds it).
- `playChunks()` — prefetches chunk *i+1*'s audio while chunk *i* is still playing, so playback doesn't stall waiting on the ~15-20s-per-chunk TTS latency.
- `fetchAudio(text, seed)` — POSTs `{model, input, seed, voice_ref, reference_text, temperature}` to `/v1/audio/speech`, returns the WAV as an ArrayBuffer, played via a `Blob` URL + `Audio` element.
- **Session seed**: `readText()` picks ONE seed per read-aloud pass (the user's fixed `settings.seed` if set, otherwise `Math.random()`-based) and reuses it for every chunk in that pass. Required because unseeded requests each sample independently — see Gotchas.
- Commands: `read-active-note`, `read-selection`, `stop-reading`, `switch-voice` (opens a `FuzzySuggestModal` voice picker). Ribbon icon `audio-lines`. Status bar shows `🔊 朗讀中 i/N`.

## audiocpp-server TTS API facts (verified by live testing, not assumed from docs)

Endpoint: `POST http://10.145.119.19:7861/v1/audio/speech`, body `{"model": "qwen3-tts", "input": "..."}` plus optional fields below. Only one model (`qwen3-tts`) is registered; `GET /v1/audio/voices?model=qwen3-tts` returns an empty list — there is no built-in named-voice picker, only zero-shot voice cloning.

**Confirmed working** (verified with a fixed `seed` + comparing output byte-for-byte across variants):
- `voice_ref` (string, **path inside the container**, not on the client) + `reference_text` (exact transcript of that wav) — this is the only real "voice/character" control. Proven live: pointing `voice_ref` at a nonexistent path returns a real `500 {"error":{"message":"could not open WAV input: ..."}}`, confirming the field is actually read per-request.
- `seed` (int) — fully deterministic; two identical requests with the same seed returned byte-identical WAVs.
- `temperature` (float) — confirmed to change output (different byte count at same seed).

**Confirmed NOT wired on this deployment** (do not add settings for these — they are silently ignored): `speaking_rate`, `pitch_shift`, `emotion`, `instruct`. Despite being real, documented flags for `audiocpp_cli`, sending them via the HTTP JSON API with a fixed seed produced byte-identical output to the baseline every time. `instruct` in particular (a CLI "voice design" flag) looked promising but tested inert on this model/route — don't rebuild this test lightly, it cost real GPU time to confirm; if the underlying `audiocpp-server` image/config is ever upgraded, it may be worth re-testing.

Without a `seed`, each request samples independently — same `voice_ref`+`reference_text`+text can still come out with audibly different tone/pacing between calls. This is *why* the plugin's session-seed mechanism above exists.

## Voice reference (container) filesystem

`/models` inside the `audiocpp-server` container is a **read-only bind mount** from `/home/hch/audio-cpp-service/models/` on the host (10.145.119.19). `docker cp` into the running container fails with `mounted volume is marked read-only`. **To add a new voice reference wav, `scp` it directly to the host path**, e.g.:

```bash
scp my_voice.wav hch@10.145.119.19:/home/hch/audio-cpp-service/models/voice_ref_<name>.wav
```

it appears inside the container automatically (still read-only from the container's side, which is fine — only read access is needed).

Existing default reference (`/models/voice_ref.wav`): 24000 Hz mono 16-bit PCM, ~4s. Match this format for new references (`ffmpeg -i in.wav -ar 24000 -ac 1 -sample_fmt s16 out.wav`), aim for ~10-20s of clean speech, and make `reference_text` an exact transcript — mismatch degrades cloning quality.

### Generating a reference clip with Windows' built-in TTS (no external tool needed)

This machine has Windows SAPI voices usable via `System.Speech.Synthesis` in PowerShell — useful for quickly producing a distinct-gender/character reference clip to feed into qwen3-tts's voice cloning:

```powershell
Add-Type -AssemblyName System.Speech
$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.GetInstalledVoices() | ForEach-Object { $_.VoiceInfo.Name, $_.VoiceInfo.Gender, $_.VoiceInfo.Culture }
# Installed as of 2026-07: Microsoft Hanhan (F, zh-TW), Microsoft Yating (F, zh-TW),
# Microsoft Zhiwei (M, zh-TW), Microsoft Zira (F, en-US)
$synth.SelectVoice("Microsoft Yating")
$synth.SetOutputToWaveFile("$env:TEMP\ref_raw.wav")
$synth.Speak("一段十秒左右、清楚自然的中文參考語音，內容盡量像正常說話。")
$synth.SetOutputToNull()
```
Then resample with ffmpeg as above before scp'ing it to the host.

Then add the preset (either via the plugin's settings UI, or by editing `data.json`'s `voicePresets` array directly — each entry is `{id, name, voiceRef, referenceText}`) and set `activePresetId`.

## Applying main.js / manifest.json edits

Obsidian must be **fully reloaded** to pick up `main.js` changes — editing the file alone does nothing to the running app. The in-app command palette has "Reload app without saving" (Ctrl/Cmd+P), but a full process restart is more reliably a clean reload:

```powershell
$obsidianPath = (Get-Process -Name Obsidian | Select-Object -First 1).Path
Get-Process -Name Obsidian | Stop-Process -Force
Start-Sleep -Seconds 2
Start-Process $obsidianPath
```

**Gotcha:** `Stop-Process -Force` while the user is actively mid-edit in the settings page can interrupt an in-flight `saveData()` disk write, leaving `data.json` on an older value than what the settings UI showed right before the kill (this happened once — user saw `chunkSize` revert). It is **not** a bug in the plugin's settings-loading (`this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData())` correctly loads and overlays `data.json` on every start) — it's a race from force-killing mid-write. Prefer confirming the settings page isn't actively being edited before force-restarting, or give a beat after a change before killing the process.

## Windows/Git-Bash CJK encoding trap when testing the API by hand

Passing Chinese text inline via `python -c "..."` (or `curl -d '...'`) through Git Bash on this machine silently mangles the UTF-8 bytes (console/argv codepage issue) — the request *looks* fine in printed output but the server sees garbage or the field is corrupted, causing misleading test results (e.g. an `instruct` field that appeared to have no effect actually had unreadable content, not zero effect — confirmed separately via nonexistent-path error testing). **Always write a `.py` file with `Write` and run `python file.py`** for any test involving non-ASCII text; never pass CJK inline as a shell argument.

## audiocpp-server operational notes

- Not always running — check first: `ssh hch@10.145.119.19 "docker ps -a --filter name=audiocpp-server"`, start with `docker start audiocpp-server` if `Exited`.
- Shares GPU 1 on the host with nothing else heavy (vllm-server uses GPU 0) but still has real latency: cold start (first request after container start) took ~38s for a 50-char sentence (CUDA graph warmup); warm throughput is roughly ~0.1s/char, i.e. ~15-20s for a 150-200 char chunk. This is why the plugin prefetches the next chunk while the current one plays instead of waiting serially.

---

## Conformance Addendum

## When to Use
Manage the "audiocpp TTS Reader" Obsidian plugin (D:\OB\.obsidian\plugins\audiocpp-tts\) that reads the active note or selection aloud via the self-hosted audiocpp-server TTS engine on 10.145.119.19:7861. Use when the user wants to edit/redeploy this plugin, add or change a TTS voice/character, debug inconsistent voice across a reading, understand why a settings-page value doesn't match what's saved, or otherwise work on Obsidian text-to-speech.

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
