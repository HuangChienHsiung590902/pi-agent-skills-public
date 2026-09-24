---
name: pipecat
description: Scaffold and run Pipecat voice-agent projects on this Windows machine via the pipecat-ai-cli (`pc`). Covers plugin marketplace setup, the broken-uv-venv gotcha, `pc init` non-interactive flow, and the critical daily-python-has-no-Windows-wheel limitation.
triggers:
  - pipecat
  - pc init
  - voice bot
  - pipecat-ai-cli
argument-hint: "[init|sync|troubleshoot]"
---

# pipecat Skill

## Purpose

Pipecat (https://pipecat.ai) is a framework for building real-time **voice conversation bots**
(STT→LLM→TTS "cascade" pipeline, or a single speech-to-speech "realtime" model). It is not a
text chatbot framework and it cannot be used for STT-only or TTS-only tasks — the CLI enforces a
full pipeline.

The `/pipecat:init` skill (from the `pipecat-ai/skills` marketplace) drives `pc init`
interactively. This skill file records the setup gotchas discovered getting it working on this
Windows box, so they don't need to be re-diagnosed next time.

## One-time setup (marketplace + plugin + CLI)

```bash
claude plugin marketplace add pipecat-ai/skills
claude plugin install pipecat@pipecat-skills          # /pipecat:init
claude plugin install pipecat-mcp-server@pipecat-skills  # /pipecat-mcp-server:talk (optional)
claude plugin install pipecat-cloud@pipecat-skills     # /pipecat-cloud:deploy (optional)
```

Restart Claude Code after installing — plugin skills only load on session start.

Install the `pc` CLI itself with `uv tool install pipecat-ai-cli` (see gotcha below before running
this — a stale/broken interpreter registration on this machine makes the first attempt fail).

## Gotcha: `uv tool install` fails with a Python venv error on this machine

Symptom on the **first** attempt:
```
error: Could not find a suitable Python executable for the virtual environment based on the interpreter: D:\python3.13\bin\python.exe
```
`D:\python3.13` is a stripped-down portable Python on this box — it's missing
`Lib\venv\scripts\nt\venvlauncher.exe`, so `venv` creation from it always fails
(`Unable to copy '...\venvlauncher.exe'`, `[WinError 2]`).

**Fix — force uv to use its own managed interpreter instead:**
```bash
uv tool install pipecat-ai-cli --python 3.13.13
```

This can still fail a **second** way — a half-created tool venv left over from the first failed
attempt (only a `Scripts\` folder with DLLs, no `pyvenv.cfg`, no `Lib\`) causes:
```
error: Querying Python at `...\uv\tools\pipecat-ai-cli\Scripts\python.exe` failed ... ModuleNotFoundError: No module named 'encodings'
```
**Fix — delete the broken tool venv and retry:**
```bash
rm -rf "$HOME/AppData/Roaming/uv/tools/pipecat-ai-cli"
uv tool install pipecat-ai-cli --python 3.13.13
```
Verify: `pc --version` should print `ᓚᘏᗢ Pipecat CLI Version: x.y.z`.

## Discovering current options

Service lists (STT/LLM/TTS/realtime/video/transports) change over time — never hardcode them.
Always run first:
```bash
pc init --list-options
```

## `pc init` validation quirks

- **Cascade mode requires all three of `--stt`, `--llm`, `--tts`.** There is no way to configure
  only STT (or only one leg of the pipeline) — `pc init` rejects it:
  ```
  Configuration validation failed:
    - --llm is required for cascade mode
    - --tts is required for cascade mode
  ```
  If the user only wants speech recognition with no LLM/TTS reply loop, Pipecat is the wrong tool —
  say so rather than forcing a fake LLM/TTS choice.
- Use `--dry-run` to validate a flag combination without generating files — cheap way to check
  requirements before running the interactive question flow to completion.
- `--output <dir>` is a *parent* directory; `pc init` still creates a `<name>/` subfolder inside
  it. Passing `--output ./my-bot --name my-bot` produces `./my-bot/my-bot/`, not `./my-bot/`. If
  the user wants the project directly at `./my-bot`, either omit `--output` (defaults to cwd +
  `<name>/`) or pass the *parent* of the intended location.

## Critical gotcha: `daily` transport cannot run its server on native Windows

`daily-python` (Daily.co's Python SDK, needed by the `daily` transport) **has never published a
Windows wheel** — confirmed by scanning all PyPI releases, every single one only ships
`manylinux_2_28_{aarch64,x86_64}` / `macosx_{10_15_x86_64,11_0_arm64}`. `uv sync` on the generated
`server/` fails immediately:
```
error: Distribution `daily-python==0.30.0 @ registry+https://pypi.org/simple` can't be installed
because it doesn't have a source distribution or wheel for the current platform
hint: You're on Windows (`win_amd64`), but `daily-python` (v0.30.0) only has wheels for ...
```

**This is not fixable by any uv flag or pip trick** — there is no Windows build. Decide *before*
running `pc init` (asking the user) which path to take if they're on native Windows (not WSL):

1. **Use WSL** for the `server/` half only — `uv sync` and `uv run bot.py` inside WSL, keep the
   `client/` (React/Vite) running natively on Windows since it's pure JS. Best if the user
   specifically wants Daily's SaaS transport (best-documented, most integrations).
2. **Switch transport to `smallwebrtc`** — self-hosted WebRTC, pure Python, works natively on
   Windows, no external SaaS account needed.
3. **Switch transport to `websocket`** — simplest, pure Python, native Windows, but audio
   framing/VAD is more DIY than the other two.

If the project was already scaffolded with `daily` before this was discovered, re-run
`/pipecat:init` with a different `--transport` rather than trying to patch the generated project —
the generated `pyproject.toml`/`bot.py` are transport-specific.

## Realtime vs Cascade — what to tell the user if they ask "just STT" or "is this a voice bot"

- **Realtime mode** (`--mode realtime --realtime <service>`, e.g. `gemini_live_realtime`,
  `openai_realtime`): one speech-to-speech model does STT+LLM+TTS internally in one hop. Lower
  latency, more natural turn-taking, but fewer vendor choices (7 total, see `--list-options`).
  There is no separately-configurable "STT" step in this mode.
- **Cascade mode**: STT → LLM → TTS as three independently swappable services. More vendor choice,
  slightly higher latency.
- Either way, the end result is a **full duplex voice conversation bot** — not a transcription
  tool, not a text chatbot. If the user's actual need is "just transcribe audio to text", point
  them at the `whisper-service` skill instead, not Pipecat.

## Server / client run steps (after successful `pc init` + working transport)

```bash
# client (separate terminal, if client-framework != none)
cd <project>/client
npm install
npm run dev

# server
cd <project>/server
uv sync
cp .env.example .env
# edit .env with the API keys for whatever STT/LLM/TTS/realtime services were chosen
uv run bot.py --transport <transport>
```

Deploying to Pipecat Cloud (if `--deploy-to-cloud` was used): see `/pipecat-cloud:deploy` skill,
or https://docs.pipecat.ai/deployment/pipecat-cloud.

## Deploying the server via Docker on a remote Linux box (10.145.119.19)

When native Windows can't run the server (`daily` transport, or any future Windows-incompatible
dependency), the working pattern on this setup is: **scaffold on Windows, run the container on
`hch@10.145.119.19`** (Linux, Docker 29.1.3, Ubuntu 24.04 — see `omc-learned/llama-cpp-server-setup.md` Step 7
for other things that run there). This does *not* need GPU, so it doesn't conflict with that box's
`gpu-switch` VRAM-sharing constraint.

### Step 1 — scaffold locally, generate `uv.lock` locally

`pc init` on Windows works fine as long as the chosen transport has no Windows-incompatible deps
(`smallwebrtc`/`websocket` are fine; `daily` is not — see gotcha above). If `pc init` doesn't leave
a `uv.lock` in `server/` (observed once, cause unclear), generate it manually — this also cheaply
re-validates the dependency set resolves on this machine before shipping it anywhere:
```bash
cd <project>/server
uv lock
```

### Step 2 — ship only the source files, not `.venv`

```bash
ssh hch@10.145.119.19 "mkdir -p ~/pipecat"
scp bot.py pyproject.toml uv.lock .env.example hch@10.145.119.19:~/pipecat/
```
Never scp `.venv/` or `.ruff_cache/` — they're platform-specific / regenerable and huge.

### Step 3 — write a *separate* Dockerfile for plain `docker run`

**The `Dockerfile` that `pc init --deploy-to-cloud` generates is Pipecat-Cloud-specific** — it's
`FROM dailyco/pipecat-base:latest` with no `CMD`, because Pipecat Cloud's own runtime injects the
entrypoint. It will build, but `docker run` on it does nothing (no process starts). Write a
separate `Dockerfile.local` instead (keep the original `Dockerfile` untouched — you may still want
`/pipecat-cloud:deploy` later):
```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

# Only needed if the chosen transport pulls in opencv-python (smallwebrtc does) — see gotcha below
RUN apt-get update && apt-get install -y --no-install-recommends \
    libxcb1 libx11-6 libxext6 libxrender1 libsm6 libglib2.0-0 libgl1 \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev
COPY bot.py .

EXPOSE 7860
CMD ["uv", "run", "--no-dev", "bot.py", "--host", "0.0.0.0", "--port", "7860"]
```
Build: `ssh hch@10.145.119.19 "cd ~/pipecat && docker build -t <name> -f Dockerfile.local ."`
(run this via `run_in_background: true`, it's a ~90s cold build — `daily-python` alone is 13.9MiB).

### Gotcha: `smallwebrtc`/webrtc transport needs `opencv-python`, which needs X11 shared libs

`bot.py` fails at import time, not at `uv sync` time — the wheel installs fine, it's a **runtime**
`ImportError`:
```
File ".../site-packages/cv2/__init__.py" ... ImportError: libxcb.so.1: cannot open shared object file
```
`python:3.12-slim`-family images don't ship X11 libs. Fix is the `apt-get install` block above
(`libxcb1 libx11-6 libxext6 libxrender1 libsm6 libglib2.0-0 libgl1`) — cheaper than switching to a
non-slim base image.

### Gotcha: the dev-runner binds to `localhost`, not `0.0.0.0` — unreachable through Docker's port mapping

Pipecat's built-in dev runner (`pipecat.runner.run.main()`, what plain `uv run bot.py` invokes)
defaults to `Uvicorn running on http://localhost:7860`. Docker's `-p 7860:7860` port mapping still
"works" (container starts, `docker ps` shows the mapping) but every request — even `curl` **from
the Docker host itself** — gets connection-refused (`curl` reports `HTTP 000`), because `localhost`
inside the container's network namespace doesn't route to the host-mapped interface. This is easy
to mistake for a firewall problem; it isn't (`ufw` isn't even installed on this box).

**Fix — pass `--host 0.0.0.0 --port <port>` explicitly** (`bot.py --help` lists these flags — every
`pc init`-generated `bot.py` accepts them, they come from the shared Pipecat runner, not
per-project code). Bake them into the Dockerfile `CMD` (see above) so it's not something to
remember to add to every `docker run` invocation. Verify with `docker logs` — the line must read
`http://0.0.0.0:<port>`, not `http://localhost:<port>`, before testing connectivity.

### Gotcha: Krisp noise filter (`--enable-krisp`) throws `ImportError` outside Pipecat Cloud

If `pc init` was run with `--enable-krisp`, the generated `bot.py` contains:
```python
if os.environ.get("ENV") != "local":
    from pipecat.audio.filters.krisp_viva_filter import KrispVivaFilter
```
`krisp_viva_filter` ships bundled in Pipecat Cloud's `dailyco/pipecat-base` image only — it is not
a normal pip dependency reachable from PyPI, so importing it in a plain `uv sync`'d container
raises `ModuleNotFoundError`/`ImportError` at bot startup. **Fix: add `ENV=local` to `.env`** to
take the `else` branch (Krisp simply isn't applied outside Cloud — this is expected, not a
workaround for a bug).

### Gotcha: `uv run` at container *runtime* re-syncs and pulls in the `dev` dependency group

Even after `uv sync --locked --no-dev` at **build** time, `CMD ["uv", "run", "bot.py"]` without
`--no-dev` makes `uv run` re-sync at **every container start**, silently downloading `pyright`
(~6MiB) and `ruff` (~11MiB) and adding ~10-90s to cold start (bytecode-compiling ~10k files each
time). Fix: pass `--no-dev` to the `CMD`'s `uv run` too (already included in the Dockerfile above)
— `uv run --no-dev bot.py ...`.

### Step 4 — run it

```bash
ssh hch@10.145.119.19 "docker run -d --name <name>-run --restart unless-stopped \
  --network host --env-file ~/pipecat/.env <name>"
```
`--restart unless-stopped` survives host reboots and container crashes without needing a systemd
unit. `--network host` is required for WebRTC transports — see gotcha below; with host networking
there's no `-p` mapping (the container binds directly to the host's `7860`). Verify end-to-end (not
just `docker ps`):
```bash
ssh hch@10.145.119.19 "docker logs <name>-run --tail 15"   # must show http://0.0.0.0:<port>
curl -s -o /dev/null -w "%{http_code}\n" http://10.145.119.19:<port>/   # 307 is normal (redirect to UI)
```

### Critical gotcha: default Docker bridge networking silently breaks WebRTC's media (UDP) path

Signaling (the HTTP `/api/offer` exchange) works fine through a normal `-p 7860:7860` port mapping
— `docker logs` shows `Connected to Gemini service`, the browser gets as far as
`ICE connection state is checking, connection is connecting`, and then **hangs forever**, eventually
logging repeated `WARNING:asyncio:socket.send() raised exception.` on the server side and
`trackStopped ... for participant undefined` in the browser console. This looks like a Gemini/API
problem but isn't — it's that ICE/RTP is a **separate UDP media path** aiortc opens on ephemeral
ports, and default Docker bridge networking NATs the container so those ports/candidates aren't
reachable from outside the container at all. Only `-p 7860:7860` (TCP, for signaling) doesn't help
here — media never gets a route.

**Fix (Linux Docker host only — this box qualifies): use `--network host` instead of `-p` mappings.**
Once on host networking, the container shares the host's real network stack directly, so the ICE
candidates it generates are actually reachable. Confirm in `docker logs` that the browser reaches
`Client READY` / `Agent READY` and the conversation UI shows the assistant's spoken response — a
`socket.send() raised exception` spam accompanied by an indefinite "Connecting..." UI state is the
signature of this specific failure, not a transient network blip worth retrying.

### Critical gotcha: `docker restart` does NOT reload `--env-file`

After editing `.env` (e.g. fixing a bad API key or model name), `docker restart <name>` brings the
*same* container process back up with the *same* environment it was originally created with —
`--env-file` is only read at `docker run`/`docker create` time, never on restart. Symptom: you fix
`.env`, restart, and the exact same error recurs verbatim in the logs, which looks like the fix
didn't take effect (or worse, like you edited the wrong file). **Fix: `docker rm -f <name> && docker
run ...` again** (recreate, don't restart) whenever `.env` changes. Verify the new values actually
landed: `docker exec <name> env | grep GOOGLE`.

### Gotcha: `pc init`'s example `.env.example` values are wrong for `gemini_live_realtime`

Two separate wrong-format traps in the generated `.env.example` comments, both surfacing only at
runtime (not at `uv sync`/build time):

- **`GOOGLE_MODEL`**: the comment suggests `"gemini-2.5-flash"`, which errors with
  `models/gemini-2.5-flash is not found for API version v1beta, or is not supported for
  bidiGenerateContent` — that's a *regular* Gemini model name, not a Live-API (bidiGenerateContent)
  one. The Live-capable model pipecat's own `GeminiLiveLLMService` defaults to internally is
  `models/gemini-2.5-flash-native-audio-preview-12-2025` (full `models/` prefix required) — grep the
  installed package for the current correct value if this default has moved on:
  ```bash
  docker run --rm <image> sh -c 'grep -n "default_settings = self.Settings" -A3 \
    /app/.venv/lib/python3.12/site-packages/pipecat/services/google/gemini_live/llm.py'
  ```
- **`GOOGLE_VOICE_ID`**: the comment suggests `"en-US-Chirp3-HD-Charon"` — that's a **Google Cloud
  TTS** voice ID format, valid for the `google_tts` *cascade* service, not for
  `gemini_live_realtime`. The Live API's prebuilt voices are short names: `Charon`, `Puck`, `Kore`,
  `Fenrir`, `Aoede`, etc. Using the Cloud-TTS-style long name doesn't error explicitly but silently
  produces no working voice — always double check against the service's own `Settings` dataclass
  default before assuming the `.env.example` comment is correct.

### Testing a WebRTC bot in a browser when it's only reachable by IP, not `localhost`

Chrome (and other browsers) disable `getUserMedia`/`RTCPeerConnection` outside a "secure context" —
`https://` or `http://localhost`/`127.0.0.1` — silently. Symptom: page loads fine, but console shows
`Failed to initialize transport: Error: WebRTC not supported or suppressed`, and the UI never gets
past a loading spinner. Browsing straight to `http://10.145.119.19:7860` trips this every time (it's
neither localhost nor HTTPS). **Fix for local testing: SSH local port-forward, then browse
`localhost`:**
```bash
ssh -L 7860:localhost:7860 -N hch@10.145.119.19 &   # background; use run_in_background: true
```
Then open `http://localhost:7860/client/` — `localhost` is always treated as a secure context
regardless of what's actually behind it. (For real end users, not just testing, put this behind
HTTPS — e.g. the [[cloudflare-tunnel]] setup already used for other services on this machine.)

### Env var naming: `GOOGLE_API_KEY`, not `GEMINI_API_KEY`

Pipecat's Google/Gemini integration (both the `google_gemini_llm` cascade service and the
`gemini_live_realtime` realtime service) reads `GOOGLE_API_KEY` from `.env` — a user handing over a
key they call "the Gemini key" or "`GEMINI_API_KEY`" is giving the same credential, just under the
name the *Google AI Studio* dashboard uses, not the env var name Pipecat's generated `.env.example`
expects. Map it across, don't add a second env var.

---

## Conformance Addendum

## When to Use
Scaffold and run Pipecat voice-agent projects on this Windows machine via the pipecat-ai-cli (`pc`). Covers plugin marketplace setup, the broken-uv-venv gotcha, `pc init` non-interactive flow, and the critical daily-python-has-no-Windows-wheel limitation.

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
