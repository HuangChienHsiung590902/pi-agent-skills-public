---
name: windows-mic-diagnose
description: Diagnose why a browser page's getUserMedia microphone isn't picking up sound on this Windows machine, even though mic permission shows "granted". Use when a user reports "麥克風收不到聲音"、"mic not picking up sound"、"沒有反應"、"沒有跳動" or a WebRTC/getUserMedia test stays silent despite the person actually speaking. Covers the JS-side check (MediaStreamTrack.muted + RMS level analysis) that tells a real hardware/OS-level mute apart from a permission or app bug, plus the Windows-side checks (Sound Control Panel recording level meter, per-app/global microphone privacy consent registry, and processes that commonly hold the mic exclusively like VoiceMeeter/OBS/Discord/Teams/Zoom).
---

# windows-mic-diagnose

## The core insight

A "granted" mic permission and a "live" `MediaStreamTrack` do **not** mean audio is actually flowing. `MediaStreamTrack.muted` is the ground truth for whether the browser is actually receiving samples — `true` combined with an RMS/level analysis showing all-zero audio is a hardware/OS-level mute, not a JS bug, permission bug, or app bug on the page's end.

Confirmed live (audiocpp-web session, 2026-07-22): permission `granted`, device correctly enumerated, `track.muted: true`, RMS peak `0` over 4 seconds while the human was speaking. Root cause turned out to be a hardware-level mute the human found and cleared themselves — re-running the same check afterward showed `track.muted: false`, RMS peak `~0.49`. The whole diagnosis, from "user reports silence" to "confirmed system-wide hardware mute, not a code bug," took about 5 tool calls.

## Step 1: JS-side check (run via browser_evaluate on the target page)

Confirms whether the browser is actually receiving audio samples, independent of anything server-side. Ask the human to speak during the 4-second sample window.

```js
async () => {
  const perm = await navigator.permissions.query({ name: 'microphone' }).catch(e => ({state: 'query-failed: ' + e.message}));
  const devices = await navigator.mediaDevices.enumerateDevices();
  const audioInputs = devices.filter(d => d.kind === 'audioinput').map(d => ({ label: d.label, deviceId: d.deviceId.slice(0,10) }));

  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  const track = stream.getAudioTracks()[0];
  const trackInfo = { label: track.label, muted: track.muted, enabled: track.enabled, readyState: track.readyState, settings: track.getSettings() };

  const ctx = new AudioContext();
  const source = ctx.createMediaStreamSource(stream);
  const analyser = ctx.createAnalyser();
  analyser.fftSize = 2048;
  source.connect(analyser);
  const data = new Uint8Array(analyser.frequencyBinCount);

  let maxLevel = 0;
  const start = Date.now();
  while (Date.now() - start < 4000) {
    analyser.getByteTimeDomainData(data);
    let sumSquares = 0;
    for (let i = 0; i < data.length; i++) {
      const v = (data[i] - 128) / 128;
      sumSquares += v * v;
    }
    const rms = Math.sqrt(sumSquares / data.length);
    if (rms > maxLevel) maxLevel = rms;
    await new Promise(r => setTimeout(r, 200));
  }

  stream.getTracks().forEach(t => t.stop());
  ctx.close();

  return { permissionState: perm.state, audioInputs, trackInfo, maxLevel };
}
```

Read the result against this table:

| `permissionState` | `track.muted` | `maxLevel` | Diagnosis |
|---|---|---|---|
| `denied` / `prompt` | — | — | Real permission problem — not this skill's territory. Fix the site permission (`chrome://settings/content/microphone` or the page's own permission UI). |
| `granted` | `true` | `0` | **Hardware/OS-level mute** — go to Step 2. |
| `granted` | `false` | `0` even while speaking | Device is open and "live" but genuinely silent — confirm it's the right physical mic, isn't obstructed, and the human actually spoke inside the 4s window (retry with an out-loud "speak now" cue). |
| `granted` | `false` | `> ~0.05` | Mic is fine. If the page still isn't working, the bug is downstream (e.g. how audio is chunked/sent), not the microphone. |

## Step 2: Windows-side checks (only if Step 1 showed `muted: true` / all-zero)

Run the bundled script — it checks for common mic-monopolizing apps, dumps the microphone privacy consent state (global + per-app), and opens Windows' own Sound Control Panel recording tab so you can screenshot it and visually confirm the OS-level meter is equally dead (proving it's system-wide, not specific to this browser tab or page).

```powershell
pwsh -File "C:\Users\HCH\.claude\skills\windows-mic-diagnose\scripts\scripts/check-windows-audio.ps1"
```

It prints the exact screenshot snippet to run next. Take two screenshots about 1 second apart while the human speaks; if the level bar next to the default recording device doesn't visibly change between them, the mute is confirmed real and system-wide.

**What the script checks, roughly most-to-least likely real-world cause:**
1. Whether a known mic-monopolizing app is running (VoiceMeeter, OBS, Discord, Teams, Zoom, Skype, Audacity, AudioRelay) — these can hold exclusive access even without appearing as their own "recording device" entry in the Sound panel.
2. Windows' per-app microphone consent registry (`HKCU:\...\CapabilityAccessManager\ConsentStore\microphone\NonPackaged\<escaped exe path>`) — an explicit `Value: Deny` here silently overrides the global Allow, with zero indication anywhere in Chrome's own UI.
3. The global microphone consent switches (`HKCU` and `HKLM`, without `\NonPackaged\...`) — both should read `Allow`.
4. Opens `control.exe mmsys.cpl,,1` (Sound → Recording tab) for the visual cross-check against the OS's own meter.

## Root causes seen so far, ranked by frequency

1. **Physical/hardware mute** (a dedicated switch, an Fn-key combo, sometimes tied to a camera-privacy shutter on business laptops). The script can't detect this directly — if everything above checks out clean (no monopolizing app, consent all `Allow`) and the OS-level meter is still dead, tell the human to look for a physical mute control. This was the confirmed cause the one time this has come up: resolved itself the instant the human found and toggled it, with zero settings/code change needed on the agent's side.
2. Windows per-app `Deny` in the consent registry — rare but silent and easy to miss since Chrome's own permission prompt still says "granted."
3. Another app holding exclusive access to the device.

## Don't confuse this with

- A **site-level permission prompt failing to appear at all**. That's a different bug — e.g. Android's "此網站無法要求你授予權限，請關閉其他應用程式的對話框或重疊視窗" happens when some OTHER app has a `SYSTEM_ALERT_WINDOW`-type overlay active (check via `adb shell dumpsys window windows | grep -iE "TYPE_APPLICATION_OVERLAY|TYPE_SYSTEM_ALERT"` and look for `appop=SYSTEM_ALERT_WINDOW` on a foreground app like a video app's floating window). That's a tap-jacking protection issue, unrelated to a muted mic.
- The `getUserMedia`-unavailable-entirely case (`navigator.mediaDevices === undefined`). That's a secure-context problem (page served over plain HTTP on a non-`localhost` origin) — `getUserMedia` itself throws/is undefined before you ever reach `track.muted`, and no amount of Windows-side checking will fix it. Fix: serve the page over HTTPS or `localhost`.

---

## Conformance Addendum

## When to Use
Diagnose why a browser page's getUserMedia microphone isn't picking up sound on this Windows machine, even though mic permission shows "granted". Use when a user reports "麥克風收不到聲音"、"mic not picking up sound"、"沒有反應"、"沒有跳動" or a WebRTC/getUserMedia test stays silent despite the person actually speaking. Covers the JS-side check (MediaStreamTrack.muted + RMS level analysis) that tells a real hardware/OS-level mute apart from a permission or app bug, plus the Windows-side checks (Sound Control Panel recording level meter, per-app/global microphone privacy consent registry, and processes that commonly hold the mic exclusively like VoiceMeeter/OBS/Discord/Teams/Zoom).

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
