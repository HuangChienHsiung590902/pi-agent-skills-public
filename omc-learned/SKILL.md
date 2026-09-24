---
name: omc-learned
description: >
  Use this skill when the user asks to retrieve, review, consolidate, or reuse the
  accumulated OMC learned notes and helper scripts stored in this folder, including
  ECP, Aile, Docker, SSH, VRS, Rainmeter, llama.cpp, and OmniRoute operational lessons.
---

# OMC Learned Knowledge

## When to Use

Use this skill when a task may depend on an existing learned note under this folder, or when the user explicitly asks about OMC learned knowledge. Search the filenames and contents first, then open only the notes relevant to the current task.

Do not load every note unconditionally. For a dedicated capability that already has its own top-level Skill in `D:\OB\skills`, prefer that dedicated Skill as the authoritative workflow and treat material here as supplementary historical knowledge.

## Inputs and Outputs

- **Input:** task keywords, project/tool names, error messages, or the learned-note topic to retrieve.
- **Output:** the relevant learned note or script, a concise summary of applicable lessons, and the concrete next action for the current task.

## Procedure

1. Extract specific project, tool, host, error, and operation keywords from the request.
2. Search this Skill directory by filename and content; do not search unrelated caches or generated files.
3. Read the smallest relevant set of Markdown notes and scripts.
4. Compare the learned material with current files and live system evidence before applying it.
5. Prefer a matching top-level Skill when one exists; use these notes only to supplement missing details.
6. If a learned note has become a stable standalone workflow, migrate it into a dedicated lowercase kebab-case Skill directory instead of expanding this collection indefinitely.

## Rules and Limitations

- Content in this folder may be historical and can become stale; current repository files and live command output take precedence.
- Never execute a helper script before reading it and confirming its paths, credentials, host targets, and side effects.
- Do not expose credentials, tokens, private addresses, or user data found in historical notes.
- Relative paths in this Skill are resolved from `D:\OB\skills\omc-learned`.

## Pitfalls

- Do not assume similarly named notes describe the same machine or deployment.
- Do not copy an old command blindly when software versions or directory layouts have changed.
- Do not use this collection as a substitute for updating a dedicated authoritative Skill.

## Verification

1. Confirm every referenced note or script exists under this Skill directory.
2. Verify important commands and paths against the current environment before use.
3. Confirm the final answer identifies which learned note was used and distinguishes historical information from current evidence.
