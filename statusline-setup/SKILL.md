---
name: statusline-setup
description: Install the Claude Code dual-account status line on Windows — dual-account quota (remaining %), context bar, git info, and live MCP/Skill chips. Handles jq dependency, script placement, settings.json wiring, and second-account setup. The fixed, ready-to-use script is bundled in this skill.
triggers:
  - statusline
  - status line
  - status bar
  - 狀態列
  - install statusline
  - setup statusline
  - statusline-setup
---

# Statusline Setup

Installs `scripts/statusline-command.sh` as a Claude Code status bar, with dual-account
support. Designed for Windows + Git Bash. **The corrected, ready-to-use script is
bundled alongside this skill** (`scripts/statusline-command.sh` in the skill directory) —
all known bugs are already baked in, so just copy it; no manual patching needed.

## What the status line shows

**Line 1 (resources):**
```
08:43  ①  my-project  [claude-opus-4-8]  ctx 58%    ①* 5h ████░░░░ 65% →14:00  7d ███░░░░░ 82%    ② 5h ░░░░░░░░ 96% →23:00  7d ░░░░░░░░ 94%
```
**Line 2 (work context):**
```
⎇ main*  +12/-3    ▶ ⚙ codegraph·explore  ✦ autopilot 3m    ⏱ codex: idle
```

- `ctx` — context window **remaining** (not used)
- `①②` — account badge (`*` = active account)
- `5h / 7d` — quota **remaining %** for **both accounts** side by side, with a
  time-to-reset bar (green = lots left, red = nearly exhausted)
- `⎇ branch*` — git branch + dirty marker
- `+N/-N` — uncommitted diff stats
- `⚙ server·tool` — most recently invoked **MCP** tool
- `✦ skill` — most recently invoked **skill**
  (two independent chips parsed from the session transcript)
  - **▶ bright** prefix = running **right now** (its `tool_use` has no `tool_result` yet)
  - otherwise a **dim age** (`5s`/`3m`/`1h`) = how long ago it last ran, so stale
    history never looks live

### How dual-account quota works
Each account writes its own quota to a shared cache
(`~/.claude-quota-cache/{1,2}.json`) on every render. The active account uses live
data; the other account is read from cache. The script auto-detects which account
is active from `$CLAUDE_CONFIG_DIR` (`~/.claude` → ①, `~/.claude-2` → ②). Both
accounts must each run at least once to populate their cache; an entry older than
2 h is dimmed (stale).

### How MCP / Skill chips work
The status-line input JSON carries `transcript_path` (the session JSONL). The
script tails it and finds the most recent `mcp__*` tool call and the most recent
`Skill` call independently. It also reads each event's `timestamp` to show an age,
and compares the last `tool_use` against the last `tool_result` to detect whether a
tool is still running (→ `▶`). No hooks and no settings changes are required.

---

## Prerequisites check

```bash
bash --version | head -1        # need ≥ 4 (Git Bash ships 5.x, OK)
jq --version 2>/dev/null || echo "MISSING"
```

---

## Step 1 — Install jq (Windows)

jq is required for JSON parsing.

```powershell
winget install --id jqlang.jq --accept-source-agreements --accept-package-agreements --silent
```

```bash
# Copy into ~/bin (already on Git Bash PATH — no shell restart needed)
mkdir -p ~/bin
cp "/c/Users/$USERNAME/AppData/Local/Microsoft/WinGet/Links/jq.exe" ~/bin/jq.exe
hash -r
jq --version    # should print jq-1.x.x
```

> **Pitfall:** winget adds jq to the *Windows* PATH, but Git Bash won't see it
> until the shell restarts. Copying into `~/bin` makes it available immediately.

---

## Step 2 — Install the bundled script

The fixed script ships with this skill. Copy it from the skill directory into the
account config dir(s):

```bash
# $SKILL_DIR = this skill's directory (where SKILL.md / scripts/statusline-command.sh live)
cp "$SKILL_DIR/scripts/statusline-command.sh" ~/.claude/scripts/statusline-command.sh
chmod +x ~/.claude/scripts/statusline-command.sh
```

> No bug-patching step is needed — the bundled script already has the single-`%`
> fix, the `exit 0` guard, and the CRLF (`tr -d '\r'`) fix for Windows jq.

---

## Step 3 — Wire settings.json

Edit `~/.claude/settings.json` — add the `statusLine` block:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash \"${CLAUDE_CONFIG_DIR:-$HOME/.claude}/scripts/statusline-command.sh\""
  }
}
```

`${CLAUDE_CONFIG_DIR:-$HOME/.claude}` resolves to each account's own copy of the
script (so account 2 runs `~/.claude-2/scripts/statusline-command.sh`).

---

## Step 4 — Dual-account setup (optional)

Second account lives in `~/.claude-2/`. Give it its own copy of the script + config:

```bash
mkdir -p ~/.claude-2
cp "$SKILL_DIR/scripts/statusline-command.sh" ~/.claude-2/scripts/statusline-command.sh
chmod +x ~/.claude-2/scripts/statusline-command.sh
```

Create `~/.claude-2/settings.json`:
```json
{
  "statusLine": {
    "type": "command",
    "command": "bash \"${CLAUDE_CONFIG_DIR:-$HOME/.claude}/scripts/statusline-command.sh\""
  }
}
```

The script auto-detects the active account from `$CLAUDE_CONFIG_DIR` and writes its
quota to `~/.claude-quota-cache/{1,2}.json` so each status line can show the
other's remaining quota.

---

## Step 5 — Verify

```bash
echo '{
  "transcript_path": "",
  "rate_limits": {
    "five_hour": {"used_percentage": 35, "resets_at": 4750244400},
    "seven_day": {"used_percentage": 18, "resets_at": 4750800000}
  },
  "context_window": {"used_percentage": 42},
  "model": {"display_name": "claude-opus-4-8"},
  "workspace": {"current_dir": "/d/myproject"},
  "session_id": "test"
}' | COLUMNS=160 bash ~/.claude/scripts/statusline-command.sh
echo "EXIT: $?"   # must be 0
```

Expected (colours stripped) — note quota numbers are **remaining**, single `%`:
```
HH:MM  ①  myproject  [claude-opus-4-8]  ctx 58%    ①* 5h ████░░░░ 65% →HH:MM  7d ███░░░░░ 82%
EXIT: 0
```

---

## Success criteria

- [ ] `jq --version` works from Git Bash without restarting the shell
- [ ] Script renders with single `%` (not `%%`) and quota shown as **remaining**
- [ ] Script exits 0 even when not in a git repo
- [ ] `~/.claude/settings.json` has `statusLine.type: "command"`
- [ ] (If dual-account) `~/.claude-2/` has both the script copy and settings.json
- [ ] `⚙`/`✦` chips appear after an MCP tool / skill has been used in the session

---

## Known pitfalls (all already fixed in the bundled script)

| Pitfall | Fix (baked in) |
|---------|----------------|
| `jq` not on bash PATH after winget install | Copy `jq.exe` to `~/bin` (Step 1) |
| Status line invisible — script exits 1 with no git dir → Claude Code drops output | `exit 0` at end of script |
| Double `%%` in account quota numbers | account-block uses single `%` (printed via `%b`) |
| Phantom leading spaces / blank chip on Windows | Windows `jq.exe` emits CRLF → pipeline ends with `tr -d '\r'` |
| Quota showed *used* instead of *remaining* | account block prints `100 - used` with `color_rem` |
| **Other account never appears / writes to `?.json`** | Account detection matched the full `CLAUDE_CONFIG_DIR` string, so Windows path forms (`C:\…\.claude-2`, trailing slash) fell through to `?`. Fixed: normalize backslashes→slashes, strip trailing slash, match on **basename** (`.claude` / `.claude-2`) |
| Remaining showed negative (e.g. `-4%`) when usage > 100% | clamp `100 - used` to `[0,100]` |
| Account 2 config dir doesn't exist | `mkdir -p ~/.claude-2` before writing (Step 4) |
| macOS bash 3 too old | N/A on Windows — Git Bash ships bash 5 |

---

## Maintaining the script

The canonical source also lives at `D:\Docs\Claude x 2\statusline\scripts/statusline-command.sh`
(with its own README). When you change it, re-bundle into this skill so future
installs stay current:

```bash
cp "D:/Docs/Claude x 2/statusline/scripts/statusline-command.sh" "$SKILL_DIR/scripts/statusline-command.sh"
```

## Optional integrations (not installed by default)

- `~/.claude/scripts/codex-statusline.sh` — Codex job status in line 2
- `~/.claude/scripts/quota-handoff-guard.py` — warns at 90% quota, prompts handoff

---

## Conformance Addendum

## When to Use
Install the Claude Code dual-account status line on Windows — dual-account quota (remaining %), context bar, git info, and live MCP/Skill chips. Handles jq dependency, script placement, settings.json wiring, and second-account setup. The fixed, ready-to-use script is bundled in this skill.

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
