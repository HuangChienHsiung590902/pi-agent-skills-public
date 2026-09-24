---
name: audiocpp-realtime-web
description: Manage the audiocpp-web browser test page (near-real-time mic-to-text + text-to-speech playback), public URL https://audiocpp.james-huang.org (Cloudflare Tunnel) / dev URL http://10.145.119.19:8861, which proxies to the existing audiocpp-server API (port 7861). Use when the user wants to redeploy this page after editing web/app.py or static/index.html, check whether audiocpp-web is running, view/adjust the compose override that wires it into the same docker network as audiocpp-server, or troubleshoot the Cloudflare Tunnel route.
---

# audiocpp-web: realtime ASR/TTS browser test page

## What this is

A FastAPI app (source in `web/`) serving one browser page:
- **Public/HTTPS (use this for actual mic recording):** `https://audiocpp.james-huang.org` — routed via the shared Cloudflare Tunnel `hch` (see skill `cloudflare-tunnel`, ingress rule points at `http://10.145.119.19:8861`, i.e. a remote origin, not `127.0.0.1`). **Gated by Cloudflare Access** — see "Access / login gate" below. Login is silent/SSO-like for a browser already signed into an allowed Google account (no visible prompt), but a fresh/incognito session will see Cloudflare's "Continue with Google" interstitial first.
- **Dev/plain-HTTP (TTS-only, no mic):** `http://10.145.119.19:8861` — no Access gate (direct to the container), works for everything except the microphone. Browsers only expose `navigator.mediaDevices.getUserMedia` in a secure context (HTTPS or `localhost`); a plain-HTTP LAN IP silently has no `mediaDevices` object at all, so `開始錄音` throws `Cannot read properties of undefined (reading 'getUserMedia')` on this URL. Always give a human the `https://` URL if they intend to actually record.

Page behavior:
- Mic recording, one fresh `MediaRecorder` per ~3s chunk (see Gotchas — this is NOT a naive `mediaRecorder.start(3000)` timeslice) -> `WS /ws/asr` (`wss://` under https, scheme is auto-derived) -> backend transcodes webm->wav via ffmpeg -> relayed to `audiocpp-server:7861/v1/audio/transcriptions` -> live transcript
- Text input -> `POST /api/tts` -> relayed to `audiocpp-server:7861/v1/audio/speech` -> playback

Design rationale and explicit v1 scope exclusions (no VAD, no progressive TTS playback, no auth): see `design.md`. Full task-by-task build log, including 3 real bugs found and fixed after the initial "done" state (webm/wav format mismatch, MediaRecorder chunk-header issue, ffmpeg-pipe-vs-file WAV size bug) and how each was root-caused: see `plan.md` (Tasks 9, 11, 12) and `.superpowers/sdd/progress.md`.

## Redeploy after editing source

```bash
bash "C:\Users\HCH\.claude\skills\audiocpp-realtime-web\scripts/deploy.sh"
```
This scp's `web/` and `docker-compose.web.yml` to `hch@10.145.119.19:~/audio-cpp-service/`, rebuilds, restarts the `audiocpp-web` container, and runs smoke tests (GET /, POST /api/tts).

## Check status

```bash
ssh hch@10.145.119.19 "docker ps --filter name=audiocpp-web"
ssh hch@10.145.119.19 "docker logs --tail 50 audiocpp-web"
```

## Run tests locally before deploying

```bash
cd "C:\Users\HCH\.claude\skills\audiocpp-realtime-web\web"
source .venv/Scripts/activate   # venv created during initial build, see plan.md Task 1
pytest -v
```

## Access / login gate (added after v1)

`design.md`'s v1 scope explicitly excluded auth — that held for the plain-HTTP dev URL. Once the public HTTPS URL went live, the user asked for a login gate (found by strangers finding the URL = free use of the underlying paid-ish GPU inference), added via **Cloudflare Access**, not app code. Full mechanism, account-level IdP setup, and the general gotchas (Chrome autofill garbage in the IdP form, `net stop cloudflared` silently no-op-ing, edge propagation delay) all live in skill `cloudflare-access-google-gate` — this section only records the app-specific facts:

- Access Application name: `audiocpp`, destination `audiocpp.james-huang.org`.
- Access policy "Allow me": Emails = `hch.new@gmail.com` OR `hch590902@gmail.com`.
- IdP: Google only (One-time PIN explicitly excluded).

## Gotchas

- `audiocpp-web` reaches `audiocpp-server` by container name (`http://audiocpp-server:7861`), which only resolves because `scripts/deploy.sh` composes both `docker-compose.yml` and `docker-compose.web.yml` together from the same directory (same compose project = same default network). Never run `docker compose -f docker-compose.web.yml up` alone on the remote host.
- This is a separate, additive deployment on top of the existing `audiocpp-server` service documented in skill `whisper-service` — don't confuse the two. `whisper-service` covers the underlying ASR/TTS engine; this skill covers the browser-facing proxy layer on top of it.
- ASR chunking is fixed-interval (3s), not VAD-based — expect a few seconds of latency per chunk boundary, and a chunk may cut a sentence mid-word. This is a known, accepted v1 limitation (see `design.md`).
- **`web/static/index.html` creates a NEW `MediaRecorder` instance for every 3s chunk** (`recordOneChunk`/`recordLoop` in the script), not one continuous recorder with `.start(3000)`. A continuous recorder only writes the WebM container's init/header segment into the FIRST emitted blob — every later blob is a headerless fragment no decoder (ffmpeg included) can parse standalone. If you ever "simplify" this back to a single `mediaRecorder.start(3000)`, ASR will silently work for chunk 1 and fail on every chunk after it with an ffmpeg `Invalid data found when processing input` error. Don't do that.
- **`app.py`'s `_webm_to_wav` writes ffmpeg's WAV output to a temp file, not `pipe:1`.** Piping WAV output to a non-seekable stdout pipe makes ffmpeg fall back to `0xFFFFFFFF` "unknown size" placeholders in the RIFF/data chunk headers (it can't seek back to patch in the real byte count once it's finished writing). `audiocpp-server` rejects that with a bare, unhelpful `500 {"error":{"message":"failed to read WAV data chunk"}}` — no indication in its logs of why. If you ever "simplify" this back to `... "pipe:1"` + read `stdout` directly from `communicate()`, every single ASR chunk will fail with that 500, even though the same audio content works fine via any file-based ffmpeg invocation (curl-testing a file-produced wav directly will mislead you into thinking the audio/model is fine — the bug is specifically pipe-vs-file, verify with a hex dump of the RIFF/data size fields if this regresses).
- **Public URL routing lives OUTSIDE this skill's directory** — it's one `hostname: audiocpp.james-huang.org` ingress rule in `C:\Users\HCH\.cloudflared\config.yml`, managed by skill `cloudflare-tunnel` (shared tunnel `hch`, also serving unrelated hostnames for other projects). If `https://audiocpp.james-huang.org` 404s after `scripts/deploy.sh` succeeds, the deploy is fine — check the tunnel config/service instead (`cloudflare-tunnel` skill's restart gotchas, especially: `net stop cloudflared` can silently no-op or hang — verify with `Get-CimInstance Win32_Process -Filter "Name='cloudflared.exe'"` that the process `CreationDate` actually advanced after a restart, don't trust `Get-Service` status alone).

---

## Conformance Addendum

## When to Use
Manage the audiocpp-web browser test page (near-real-time mic-to-text + text-to-speech playback), public URL https://audiocpp.james-huang.org (Cloudflare Tunnel) / dev URL http://10.145.119.19:8861, which proxies to the existing audiocpp-server API (port 7861). Use when the user wants to redeploy this page after editing web/app.py or static/index.html, check whether audiocpp-web is running, view/adjust the compose override that wires it into the same docker network as audiocpp-server, or troubleshoot the Cloudflare Tunnel route.

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
