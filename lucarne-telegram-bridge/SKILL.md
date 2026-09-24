---
name: lucarne-telegram-bridge
description: Install, configure, and troubleshoot Lucarne (lucarned) — a Telegram/WeChat bridge that remote-controls local coding agents (pi, Claude Code, Codex, Gemini, Copilot, Grok). Use when the user wants to install/reinstall lucarned, fix "the chat is not a forum" errors, set up a Telegram group with Topics, promote the bot to admin, or manage the Windows autostart service.
triggers:
  - lucarne
  - lucarned
  - pi telegram
  - telegram coding agent bridge
  - the chat is not a forum
argument-hint: "[install|fix-forum-error|status|restart]"
---

# Lucarne Telegram Bridge Skill

## Purpose

Lets you drive a local pi (or Claude Code / Codex / Gemini / Copilot / Grok) coding
session from Telegram on your phone. `lucarned` runs locally, watches your agent
session history, and creates one Telegram **Forum Topic per workspace/session** so
multiple conversations don't collide.

Upstream project: https://github.com/tuchg/Lucarne (Rust, native Windows/macOS/Linux)

## Current state on this machine (HCH / Windows)

- Bot username: **@James590902_bot**
- Binary: `C:\Users\HCH\.lucarne\bin\lucarned.exe` — **this is now an NTFS junction**
  to `D:\.system\.lucarne` (moved 2026-08-15 per the `dotfiles-to-d-system` skill so
  it survives Deep Freeze). Same for config/state — see below. Both junctions were
  created with `lucarned.exe` stopped first, then `lucarned autostart start` again
  after; doctor should show `ok` for every line with no path changes needed anywhere
  else, since junctions keep every old path working transparently.
- Config: `C:\Users\HCH\AppData\Local\lucarned\lucarned.yaml` — junction to
  `D:\.system\lucarned\lucarned.yaml` (the whole `AppData\Local\lucarned` folder is
  the junction target, not just the yaml)
- Logs: `C:\Users\HCH\AppData\Local\lucarned\logs\lucarned.YYYY-MM-DD.log` — **not**
  `~/.lucarned/logs` (that path doesn't exist on this machine; this build puts
  everything per-user under `%LOCALAPPDATA%\lucarned`, not a home dotfile — check
  `lucarned doctor` output for the real paths before assuming the generic
  `~/.lucarned` paths below apply)
- State DB: `C:\Users\HCH\AppData\Local\lucarned\state.sqlite3`
- Entry chat: a **Telegram supergroup** named "Lucarne控制台" with Topics enabled,
  chat id `-1003885611291` (NOT the private 1:1 chat — see "the chat is not a forum" below)
- Autostart: Windows Task Scheduler task `LucarneLucarned` (installed via
  `lucarned autostart install --start`, requires admin PowerShell to create).
  Trigger type is `MSFT_TaskLogonTrigger` — fires on user logon, not a persistent
  background service, so it needs an actual interactive logon (not just machine boot)
  to start.
- Agents enabled in config: `claude, codex, copilot, gemini, pi, grok`. **`/aN`
  numbering is not stable — do not hardcode "`/a2` = pi" or similar.** It appeared
  to be `/a1`=claude, `/a2`=pi for a while (both CLIs `ok` in `doctor`, the other
  four `warn`/never even register — see "`/aN` numbering is unstable" gotcha below
  for the full story of why guessing the number is a trap), then later in the same
  session `/a2` started failing with `handler error error=/a2 out of range` with no
  config change in between. **Always send `/panel` first and read the numbers it
  actually shows right now** rather than reusing a number from earlier in the
  conversation or from this doc.

## Fresh install (Windows)

1. Install the binary — **must run in a real Windows PowerShell/pwsh, not
   Git-Bash/MSYS** (MSYS mangles the installer's path handling and breaks `tar`):
   ```powershell
   powershell -NoProfile -ExecutionPolicy Bypass -Command "irm https://github.com/tuchg/Lucarne/releases/latest/download/lucarned-installer.ps1 | iex"
   ```
   Installs to `%USERPROFILE%\.lucarne\bin\lucarned.exe`.

2. Get a Telegram bot token from [@BotFather](https://t.me/BotFather) (`/newbot`).

3. **Create a Telegram supergroup with Topics enabled — do not use a private 1:1
   chat as entry_chat_id.** See "Forum Topic requirement" below for why. Steps:
   - Create a new Telegram group, add the bot to it
   - Group Info → Edit (pencil icon) → toggle **Topics** ON
     (this auto-upgrades a Basic Group to a Supergroup)
   - Group Info → Edit → Administrators → Add Admin → select the bot
     (bot needs admin rights to create/manage topics)
   - Send any message in the group's `# General` topic, then read it back with:
     ```bash
     curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates"
     ```
     The group chat id is the negative number in `"chat":{"id":-100XXXXXXXXXX, ...}`.
     **Stop lucarned first** (`taskkill /F /IM lucarned.exe`) if getUpdates returns
     empty — it long-polls the same token and will swallow updates before you can
     read them.

4. Write `%LOCALAPPDATA%\lucarned\lucarned.yaml`:
   ```yaml
   agents:
     - claude
     - pi
   state:
     db: ~/.lucarned/state.sqlite3
   logging:
     filter: "info,lucarne=info,lucarned=info"
     stderr_filter: warn
     dir: ~/.lucarned/logs
     max_files: 16
   turn:
     inactivity_secs: 1800
     deadline_secs: 3600
   session:
     idle_timeout_secs: 7200
   config:
     global:
       bypass: false
       notifications: true
   updates:
     enabled: true
     notify: true
     check_interval_hours: 24
     remind_interval_hours: 24
     repository: tuchg/Lucarne
   channels:
     telegram:
       enabled: true
       token: "<BOT_TOKEN>"
       entry_chat_id: "<GROUP_CHAT_ID>"   # the negative supergroup id, not your personal id
   ```
   (`lucarned init` is the official interactive path but requires a real TTY —
   it errors with `"run \`lucarned init\` in an interactive terminal"` when driven
   non-interactively. Writing the YAML directly works fine and is what this skill does.)

5. Verify:
   ```bash
   lucarned doctor      # checks config/logs/autostart/agent CLIs
   ```
   `pi` shows `ok` if `pi.CMD` is on PATH; `claude` shows `ok` if `claude.EXE` is found.

6. Register Windows autostart (**requires an elevated/Administrator PowerShell** —
   `schtasks` fails with "Access is denied" otherwise):
   ```powershell
   lucarned autostart install --start
   ```
   Confirm with `lucarned autostart status` (Task Scheduler task `LucarneLucarned`,
   status `Ready`).

7. In Telegram, open the group and send `/panel`. Tap `/a1` (or send it as text) to
   spin up a new `pi` agent workspace — this creates a Forum Topic named `pi · new`.

## Forum Topic requirement — "the chat is not a forum"

Lucarne creates one **Telegram Forum Topic** per agent session/workspace so
concurrent conversations don't interleave. This only works if `entry_chat_id`
points at a chat with Topics support:

- ✅ A **supergroup with Topics enabled** (`is_forum: true` in the Bot API chat object)
- ❌ A plain private 1:1 chat with the bot (unless the bot itself has private-chat
  topics via Bot API 9.4+ `has_topics_enabled`, which is not reliably available —
  check with `getMe` and look for `"has_topics_enabled": true`; it was `false`
  for @James590902_bot at setup time)

**Symptom in logs** (`%USERPROFILE%\.lucarned\logs\lucarned.YYYY-MM-DD.log`):
```
WARN lucarne_telegram: create_forum_topic failed error=channel transport: Bad Request: the chat is not a forum
WARN lucarne_telegram::bot: core event handler error error=channel transport: Bad Request: the chat is not a forum
```
When this happens every `/panel` interaction just re-shows the management panel
help text and no agent session ever actually starts — text you send never reaches
the agent.

**Fix:** switch `entry_chat_id` to a Topics-enabled supergroup id (see install
step 3), update the yaml, then:
```bash
taskkill /F /IM lucarned.exe
lucarned autostart start
```

## Bot needs Admin rights in the group

Even with Topics enabled, the bot needs **Administrator** rights in the group to
create/manage topics — a plain member shows "has no access to messages" and topic
creation still fails. Group Info → Administrators → Add Admin → select the bot.

## Old forum topics can silently lose their agent binding

A Forum Topic that worked fine before can start showing this on every message,
even years-old topics that previously had real conversations in them:
```
This workspace isn't bound to an agent session yet. Tap a history entry in the
entry panel to rebind.
```
Log signature:
```
WARN handle_message{workspace="<thread_id>" ...}: lucarne_telegram::bot: unbound
topic received a message chat=... topic=<thread_id>
```
This happened here to a `pi · new` topic that had genuine multi-turn history from
days earlier — the binding was gone even though the topic itself (and its old
messages) were untouched. Root cause not fully isolated, but it correlates with
`lucarned` restarts/config edits in between — the control-plane state
(`state.sqlite3`) presumably lost the workspace↔topic mapping for that thread at
some point without deleting the topic itself.

**If a user reports "I typed in the topic I always use and got no reply," check
this before anything else** — grep today's log for `unbound topic` with the
thread id from their screenshot. Fix: either tap a history entry in the panel to
rebind that exact topic, or (simpler) just send `/panel` → pick a fresh agent slot
to open a brand-new topic and use that one instead of fighting to rebind the old one.

## `/aN` numbering is unstable — don't hardcode it

`/aN` picks the Nth agent slot for a **new** session, but which agent that resolves
to is not a fixed function of the yaml's `agents:` list order, and it can change
mid-session with no visible config change. Observed on this machine in one
uninterrupted session:
- Provider **registration** (log: `registering provider provider_id=...`) only ever
  showed 5 of the 6 configured agents — `copilot` never registered at all, not even
  as unavailable. Registration order was `claude, codex, gemini, pi, grok`.
- `lucarned doctor` shows `ok` only for `claude` and `pi` (CLI found on PATH); `codex`,
  `gemini`, `copilot` show `warn` (not found); `grok` isn't checked by `doctor` at all.
- Early in the session, `/a1` reliably created a `claude` workspace and `/a2`
  reliably created a `pi` workspace (confirmed via
  `lucarne::agent_runtime` / `workspace upserted ... provider_id=pi` log lines) —
  consistent with "only CLI-available agents count, in registration order."
- Later in the **same session, same yaml, same running process** (no restart, no
  config edit), `/a2` started failing:
  ```
  INFO ...: entry command action=Agent(2)
  WARN lucarne_telegram::bot: handler error error=/a2 out of range
  ```
  No corresponding log line explains why the valid range shrank. It was not caused
  by deleting old forum topics (topic deletion happened chronologically first and
  `/a2` still worked afterward at that point) — the actual trigger wasn't isolated.

**Practical takeaway:** never assume a previously-working `/aN` number still works.
Send `/panel` immediately before using `/aN` and read the numbers/labels it renders
at that moment, or use the panel's tap-buttons instead of typing the digit blind.
If `/aN` errors with `out of range`, that's not a sign anything is broken — just
that the number moved; re-check via `/panel`.

## Silent no-op: `channels.telegram` left blank after an interrupted `init`

If `lucarned init` gets aborted partway (e.g. it errors out at chat discovery — see
"the chat is not a forum" below, or the "Send a message to the bot" prompt times out
before you actually message it), it can leave `lucarned.yaml` with the channel
block still at its zero-value defaults:
```yaml
channels:
  telegram:
    enabled: false
    token: ""
    entry_chat_id: null
```
`lucarned` does **not** error on this — it starts, logs a single line, and exits
cleanly with no crash and no retry:
```
INFO lucarned: no adapters enabled; edit lucarned config to enable a channel config_path=Some("...")
```
This is easy to miss because `lucarned autostart start` / `tasklist` right after
looks like it worked (task ran, exit code 0) — the process just isn't there a few
seconds later because it had nothing to do and quit. **Always tail the log** after a
fresh `autostart start` and confirm you see `telegram adapter started`, not `no
adapters enabled`. Fix: manually fill in `enabled: true`, the real bot token, and
`entry_chat_id` (see install step 4), then `taskkill /F /IM lucarned.exe` +
`lucarned autostart start` again.

## Testing the pipeline: sending as the bot doesn't count

To verify text actually reaches the agent, **don't** use
`curl .../sendMessage` with the bot's own token to post a test message into a topic
— Telegram never delivers a bot's own outgoing messages back through that same
bot's `getUpdates` long-poll, so `lucarned` will never see it and nothing happens
(no log entry at all for that message, not even a filtered/ignored one). This looks
identical to "the bridge is broken" if you don't know the cause.

The only valid test is typing from the **real Telegram user account** (the human
member of the group). A real turn completing successfully shows in the log as:
```
INFO handle_message{...}: lucarne_telegram::bot: agent turn completed elapsed_ms=8726
```
If you need to drive this via ADB automation instead of a person typing, see
`phone-termux-remote`'s gotchas on why **USB beats the ZeroTier-TCP `adb -s
<ip>:5555` transport** for `input tap` reliability — taps sent over the slower TCP
transport on this phone were repeatedly misread as swipes/long-presses in
Telegram's topic list, especially since the list live-reorders (see next section).

## The `agent notifications` topic mirrors the current coding-agent session, not just pi

`lucarned` auto-creates (or recreates, if its state was reset — e.g. after a config
edit that changed `entry_chat_id`) a Forum Topic literally named "agent
notifications" that live-mirrors whatever the **coding agent driving this very
lucarned setup session** (e.g. Claude Code, if you're using it to configure lucarne)
is saying, via a `history session watch` on that agent's own session file — this is
a different feed from the per-workspace `claude · new` / `pi · new` topics that hold
actual user↔agent conversations. Two side effects worth knowing:
- It reorders to the top of the Telegram topic list every time the mirrored
  session (e.g. this very conversation) produces new output — if you're navigating
  the topic list by tapping row positions right after asking Claude/pi a question,
  the list can reorder out from under your tap between screenshot and tap. Use
  Telegram's search (magnifying glass) to jump to a topic by name instead of
  position-based taps when this channel is active.
- If a fresh "agent notifications" topic gets created (new thread id) while an
  older one with the same name still exists from a prior session, you'll see two
  topics named identically in the list — this is expected, not a bug; the old one
  still holds real historical content and shouldn't be deleted blindly.

## Admin cleanup via Bot API — faster and safer than tapping in the app

Tapping through the Telegram app via ADB to delete test topics/messages is slow and
error-prone (multi-select mode can trigger accidentally — double-check the header
doesn't show a selection count before tapping anything that looks like a trash
icon). Prefer the Bot API directly with `curl`, since the bot is already Admin in
the group:
```bash
# delete an entire forum topic (all its messages) — irreversible
curl -s -X POST "https://api.telegram.org/bot<TOKEN>/deleteForumTopic" \
  -d "chat_id=<CHAT_ID>" -d "message_thread_id=<THREAD_ID>"

# delete a single message by id
curl -s -X POST "https://api.telegram.org/bot<TOKEN>/deleteMessage" \
  -d "chat_id=<CHAT_ID>" -d "message_id=<MESSAGE_ID>"
```
Thread IDs for topics *you* just created are printed in the log right after
creation (`lucarne_telegram: forum topic created thread_id=NNN`) — grep
`forum topic created` in today's log to get a clean list instead of hunting through
the UI. For stray individual messages sent by a real user (not via a `sendMessage`
call you made yourself, so you don't have the id from an API response), message ids
are sequential per-chat across all topics — bound the search using known ids from
nearby `last_message_id=` log lines before/after the timestamp in question, and
`deleteMessage` is safe to try-and-check since it just returns `ok:false` on a
nonexistent id rather than affecting anything else.

`lucarned` itself notices when a topic it knows about gets deleted externally and
posts a summary to `agent notifications` — that's confirmation the deletion
succeeded and its internal state tracking didn't break, not a bug to chase.

## Daily operations

```bash
lucarned doctor              # health check: binary, config, logs dir, autostart, agent CLIs
lucarned autostart status    # Task Scheduler task status
lucarned autostart start     # (re)start the service
lucarned autostart stop
lucarned paths               # print resolved config/log/db paths
lucarned update              # check for newer lucarned release
```

Manual foreground run (useful when `autostart start` needs testing before an
elevated session is available):
```bash
lucarned          # runs in foreground, Ctrl+C to stop
```
Only run one instance at a time — a second instance racing the same bot token
causes `Conflict: terminated by other getUpdates request` in the logs.

## Telegram commands (inside the entry group)

| Command | Effect |
|---|---|
| `/panel` | Show/refresh the management panel |
| `/aN` | Create/open agent workspace N (e.g. `/a1`) — opens a new Forum Topic |
| `/hN` | Open history session N from the picker |
| `/wN` | Open workspace N from the picker |
| `/new` | Start a fresh conversation in the current topic |
| `/status` | Show current turn/session status |
| `/interrupt` | Stop current agent work |
| `/fork` | List/branch fork targets |
| `/kill all\|<id:pid>` | Kill managed agent processes |
| `/quit` | Close the live session |

Inside a topic, plain text messages are forwarded to the bound agent (pi/Claude/etc).

## Known transient error: 503 chat_admission_busy

Model/provider backend capacity error, unrelated to Lucarne itself:
```
⚠️ 失敗 · 503: {"message":"Structurally heavy chat request capacity is busy; retry shortly.",
"type":"server_error","code":"chat_admission_busy","reason":"structure_limit"}
```
Log line looks like:
```
WARN ...: provider signalled turn failure event="turn_failed" ... error=503: {...chat_admission_busy...}
INFO ...: observed close reason from wrapped session provider_id=pi reason=pi exited with code -1073741510
```
The underlying `pi` session process exits when this happens — it does not silently
retry. **Fix:** send `/new` in the topic to start a fresh session, then re-ask the
question. If it recurs repeatedly, try switching model with `/model` inside the topic.

## Removal

```bash
lucarned autostart uninstall --stop     # needs elevated shell
rm -rf ~/.lucarne "%LOCALAPPDATA%\lucarned"
```
On this machine both of those are junctions to `D:\.system\.lucarne` and
`D:\.system\lucarned` — deleting the junction (not `-Recurse` through it) leaves the
real data in `D:\.system` untouched; delete that separately if you actually want the
data gone, not just the app uninstalled.

---

## Conformance Addendum

## When to Use
Install, configure, and troubleshoot Lucarne (lucarned) — a Telegram/WeChat bridge that remote-controls local coding agents (pi, Claude Code, Codex, Gemini, Copilot, Grok). Use when the user wants to install/reinstall lucarned, fix "the chat is not a forum" errors, set up a Telegram group with Topics, promote the bot to admin, or manage the Windows autostart service.

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
