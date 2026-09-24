---
name: opencode-telegram-bot
description: Install, configure, and manage @grinev/opencode-telegram-bot — a Telegram client that remote-controls a local OpenCode (opencode CLI) instance. Use when the user wants to install/reinstall the bot, add/switch a model provider, reconfigure the whitelist, or set up/verify boot auto-start.
triggers:
  - opencode telegram bot
  - opencode-telegram
  - telegram bot for opencode
  - opencode 手機
argument-hint: "[install|autostart|status|switch-model]"
---

# opencode-telegram-bot Skill

## Purpose

Lets you drive a local OpenCode coding-agent session from Telegram on your phone.
The bot talks to a local `opencode serve` (default `http://localhost:4096`) — no
open ports, nothing leaves the machine except the Telegram chat itself.

Upstream project: https://github.com/grinev/opencode-telegram-bot
(npm package: `@grinev/opencode-telegram-bot`)

## Current state on this machine (HCH / Windows)

- Bot username: **@JamesHuangBot**
- Config: `%APPDATA%\opencode-telegram-bot\.env`
- Logs: `%APPDATA%\opencode-telegram-bot\logs\`
- Model provider: `deepseek` / `deepseek-chat` (switch anytime in-chat with `/model`)
- DeepSeek key lives in the permanent user env var `DEEPSEEK_API_KEY` (`setx`), read
  by whichever process spawns `opencode serve`
- Autostart scheduled task: `OpenCodeTelegramBot-AutoStart` (fires 30s after logon)

## Fresh install / reinstall

Run `scripts/install.ps1`. It npm-installs the package globally, writes the `.env`,
makes sure `opencode serve` is running, and starts the bot daemon. Needs two things
from the user first:

1. **Bot token** — from @BotFather in Telegram (`/newbot`)
2. **Numeric user ID** — from @userinfobot in Telegram (this becomes the
   `TELEGRAM_ALLOWED_USER_ID` whitelist; the bot ignores everyone else)

```powershell
./scripts/install.ps1 -BotToken "<token>" -UserId "<numeric id>" -Locale zh
```

To wire up a paid/non-default model provider (e.g. DeepSeek) in the same pass:

```powershell
./scripts/install.ps1 -BotToken "<token>" -UserId "<numeric id>" -Locale zh `
    -ModelProvider deepseek -ModelId deepseek-chat `
    -ProviderApiKeyEnvName DEEPSEEK_API_KEY -ProviderApiKeyValue "<key>"
```

Run `opencode models` first to see what's actually available/authenticated before
picking `-ModelProvider`/`-ModelId`.

## Adding/switching a model provider's API key

**Do not try to script `opencode auth login`.** It's a full-screen TUI
(@clack/prompts) that reads raw keypresses. Piped stdin *does* work for the
provider search/select step, but the free-text "Enter your API key" step silently
drops the input and comes back "Required" no matter how the bytes are paced —
confirmed not fixable with `sleep`-staggered `printf`. `winpty` is on this machine
if a real PTY approach is ever worth revisiting, but it wasn't needed.

**Instead: use env-var auto-detection.** opencode recognizes `<PROVIDER>_API_KEY`
style env vars without any `auth.json` entry at all — confirmed for
`DEEPSEEK_API_KEY` (`opencode auth list` shows it under an "Environment" section).
So to add a provider:

```powershell
setx <PROVIDER>_API_KEY "<key>"      # e.g. DEEPSEEK_API_KEY, OPENAI_API_KEY
```

then restart whichever `opencode serve` process should pick it up (setx only
affects *future* processes, not the currently running one — kill and relaunch it,
or just reboot / let the scheduled task relaunch it next logon). Then update
`OPENCODE_MODEL_PROVIDER` / `OPENCODE_MODEL_ID` in the bot's `.env` and restart the
bot daemon (`opencode-telegram stop` then `start --daemon`), or just switch models
live from inside the Telegram chat with `/model`.

## Boot auto-start

Run `scripts/setup-autostart.ps1` once. It registers a per-user Scheduled Task
(`OpenCodeTelegramBot-AutoStart`) that runs `opencode-telegram start --daemon` 30
seconds after logon. **No separate task is needed to launch `opencode serve`** —
the bot's `.env` has `OPENCODE_AUTO_RESTART_ENABLED=true`, so on startup the bot
itself spawns a local `opencode serve` if one isn't already listening on the
configured port (see `dist/opencode/process.js` in the installed package —
`startLocalOpencodeServer` does a detached `spawn`, inheriting the daemon's own
process env, which is why the provider API key needs to be a *permanent* env var
set via `setx`, not just exported for one shell).

Verify without waiting for a reboot:

```powershell
Start-ScheduledTask -TaskName "OpenCodeTelegramBot-AutoStart"
Start-Sleep -Seconds 5
opencode-telegram status
```

## Day-to-day management

```powershell
opencode-telegram status          # PID, uptime, log file path
opencode-telegram stop
opencode-telegram start --daemon  # or `start` alone for foreground/debug
```

```powershell
# Disable/remove autostart:
Disable-ScheduledTask -TaskName "OpenCodeTelegramBot-AutoStart"
Unregister-ScheduledTask -TaskName "OpenCodeTelegramBot-AutoStart" -Confirm:$false
```

## Gotchas

- `.env.example` (in the installed package, e.g.
  `%APPDATA%\npm\node_modules\@grinev\opencode-telegram-bot\.env.example`) is the
  authoritative list of every supported env var (STT/TTS providers, scheduled-task
  limits, directory-browser roots, etc.) — read it before assuming a setting
  doesn't exist.
- `OPENCODE_MODEL_PROVIDER`/`OPENCODE_MODEL_ID` in the bot's `.env` just tell the
  bot which *already-authenticated* opencode model to default to — they don't
  carry any credentials themselves. Credentials live in opencode's own
  `~/.local/share/opencode/auth.json` or in provider env vars, never in the bot's
  `.env`.
- Never write a real bot token or provider API key into this skill's files —
  they're per-user secrets, passed as script parameters at run time only.

---

## Conformance Addendum

## When to Use
Install, configure, and manage @grinev/opencode-telegram-bot — a Telegram client that remote-controls a local OpenCode (opencode CLI) instance. Use when the user wants to install/reinstall the bot, add/switch a model provider, reconfigure the whitelist, or set up/verify boot auto-start.

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
