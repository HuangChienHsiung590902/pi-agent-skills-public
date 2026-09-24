---
name: omc-update
description: Use when omc-doctor reports plugin outdated, OMC skills disappear after restart, or after clearing the plugin cache and skills fail to reload. Do NOT follow omc-doctor's "Fix 1" (cache clear only) — it leaves a stale marketplace and Claude Code re-downloads the old version.
---

# OMC Plugin Update

## The Trap

`omc-doctor` Fix 1 clears the cache but NOT the marketplace. On restart, Claude Code sees the entry in `installed_plugins.json` is still valid and re-downloads the **same old version** from the stale marketplace snapshot (cloned weeks ago). Skills appear to load but are at the old version.

## Correct Update Flow

```powershell
# 1. Update the marketplace git clone to latest
claude plugin marketplace update omc

# 2. Update the plugin from the refreshed marketplace
claude plugin update oh-my-claudecode@omc

# 3. Restart Claude Code
```

## If Skills Disappear After Cache Clear

Cache was cleared but `installed_plugins.json` still has the old record → Claude Code re-registers without downloading files:

```powershell
# Force a clean reinstall
claude plugin uninstall oh-my-claudecode
claude plugin install oh-my-claudecode

# Then update marketplace + plugin (steps above)
claude plugin marketplace update omc
claude plugin update oh-my-claudecode@omc
```

## Verify

```powershell
claude plugin list
# Should show oh-my-claudecode@omc at the new version
```

Then restart. If `oh-my-claudecode:omc-doctor` appears in skills list → done.

---

## Conformance Addendum

## When to Use
Use when omc-doctor reports plugin outdated, OMC skills disappear after restart, or after clearing the plugin cache and skills fail to reload. Do NOT follow omc-doctor's "Fix 1" (cache clear only) — it leaves a stale marketplace and Claude Code re-downloads the old version.

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
