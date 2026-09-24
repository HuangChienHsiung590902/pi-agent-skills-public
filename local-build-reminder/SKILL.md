---
name: local-build-reminder
description: Remind the user to rebuild OMC after editing TypeScript when running from a local fork. Triggered automatically by the AI whenever it notices it (or the user) just changed a src/**/*.ts file in an OMC dev install.
level: 1
---

# Local Build Reminder

**Always-on reminder for OMC fork development.** When OMC is running in local
mode (HUD shows `[OMC#X.Y.ZL]` with an `L` suffix), Claude Code loads compiled
JavaScript from `dist/` — NOT TypeScript source from `src/`. Edits to `.ts`
files are invisible to the running plugin until `npm run build` regenerates
`dist/`.

## When to invoke this skill

The AI should mention this reminder whenever **any of these** happens:

1. The user (or the AI itself) just edited `src/**/*.ts` in this repo.
2. The user asks "why isn't my change working?" / "I edited X but it does the same" after a TS edit.
3. The user is about to restart Claude Code and the working tree has TS edits with no rebuild.
4. The user runs an OMC command and expects new behavior tied to a TS edit.

## What to say

Surface one clear sentence followed by the exact command. Don't repeat the
reminder on every turn — once per "round" of TS editing is enough. Example:

> Heads up: you edited `src/...`. Run `npm run build` before restarting
> Claude Code — `dist/` won't reflect the change otherwise.

If multiple TS files were edited in a row, just remind once at the end.

## When NOT to remind

- The user only edited `.mjs` / `.cjs` / `.md` / `.json` — those load directly
  from disk, no build needed.
- The user is in a Claude Code session that isn't running OMC locally
  (no `L` in the HUD).
- A `tsc --watch` / `npm run dev:full` is already running in the background
  — those rebuild automatically on save.
- The user just asked an unrelated question; don't shoehorn the reminder
  into off-topic responses.

## File-type cheat sheet

| Path                           | Restart picks up edit? | Needs build? |
| ------------------------------ | ---------------------- | ------------ |
| `src/**/*.ts`                  | Only after build       | **Yes**      |
| `templates/hooks/**/*.mjs`     | Yes                    | No           |
| `scripts/**/*.mjs` / `*.cjs`   | Yes                    | No           |
| `skills/**/SKILL.md`           | Yes                    | No           |
| `agents/**/*.md`               | Yes                    | No           |
| `commands/**/*.md`             | Yes                    | No           |
| `.claude-plugin/plugin.json`   | Yes (on Claude restart)| No           |
| `docs/**/*.md`                 | Cosmetic only          | No           |

## One-command setup for hands-free dev

If the user is iterating heavily and tired of remembering the build, suggest:

```powershell
npm run dev:full
```

This runs `tsc --watch` plus all bridge builders in parallel — every save
triggers a rebuild within a second, so `restart Claude Code` is all that's
needed afterwards.

## Detection signal — how the AI knows it's "local mode"

The HUD's `[OMC#X.Y.ZL]` suffix is the visible cue. Programmatically, the
detection lives in `src/lib/version.ts::isRuntimePackageLocal()` and triggers
on any of: `.git/` at package root, `src/` at package root, package reached
via symlink/junction, or any ancestor is a symlink/junction.

When running inside the OMC fork repo itself, the AI is by definition in
local mode — the reminder always applies.

---

## Conformance Addendum

## When to Use
Remind the user to rebuild OMC after editing TypeScript when running from a local fork. Triggered automatically by the AI whenever it notices it (or the user) just changed a src/**/*.ts file in an OMC dev install.

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
