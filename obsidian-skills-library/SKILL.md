---
name: obsidian-skills-library
description: Treat the user's Obsidian vault (`D:\OB`) as the canonical skills library, and use the official Obsidian CLI to search, open, create, inspect, and maintain `D:\OB\skills` skill folders in Skill Creator format. Use when the user asks to add/update/search/list skills, repair the Pi/Claude skills link, or operate the skills knowledge base through Obsidian instead of scattered agent-local folders.
---

# Obsidian Skills Library via Obsidian CLI

## When to Use
Use this when the user wants Obsidian to be the single source of truth for agent skills, especially when they ask to:

- create a new skill in `D:\OB\skills`
- search existing skills before answering or editing
- open a skill in Obsidian for review
- update `SKILLS_INDEX.md`
- repair or verify the Pi/Claude skills path after files were deleted
- keep skills in the Skill Creator standard format rather than ad-hoc notes

This complements `obsidian-cli`: that skill explains the raw CLI commands; this skill defines the workflow and conventions for using the Obsidian vault as the skills library.

## Procedure
1. Confirm the canonical locations before editing:
   - Vault root: `D:\OB`
   - Skills source of truth: `D:\OB\skills`
   - Index file: `D:\OB\skills\SKILLS_INDEX.md`
   - Pi agent configured skills path: `D:\.system\.claude\skills`
   - Expected junction target: `D:\.system\.claude\skills => D:\OB\skills`
2. Confirm Obsidian CLI is available and the vault is open. Prefer PowerShell on Windows:
   ```powershell
   & 'C:\Program Files\Obsidian\Obsidian.exe' vault 2>&1 | Out-File -Encoding utf8 -FilePath "$env:TEMP\obsidian-vault.txt"
   Get-Content -Raw "$env:TEMP\obsidian-vault.txt"
   ```
   Expected result includes `name OB` and `path D:\OB`. If Obsidian is not running, start/open Obsidian first; the CLI is a remote controller, not a headless vault parser.
3. Search for existing skills before creating a new one, to avoid duplicates:
   ```powershell
   & 'C:\Program Files\Obsidian\Obsidian.exe' search query='關鍵字' --json
   & 'C:\Program Files\Obsidian\Obsidian.exe' search:context query='關鍵字' context=3
   ```
   Also search the filesystem when exact names matter:
   ```powershell
   Get-ChildItem 'D:\OB\skills' -Directory | Where-Object Name -like '*keyword*'
   Select-String -Path 'D:\OB\skills\*\SKILL.md' -Pattern 'keyword' -CaseSensitive:$false
   ```
4. Read/open existing skills through Obsidian CLI when the user wants Obsidian-centric operation:
   ```powershell
   & 'C:\Program Files\Obsidian\Obsidian.exe' read path='skills/obsidian-cli/SKILL.md'
   & 'C:\Program Files\Obsidian\Obsidian.exe' open path='skills/obsidian-cli/SKILL.md' newtab
   ```
   For large or precise edits, still use normal file tools against `D:\OB\skills\...\SKILL.md`, then use the CLI to open/read/search for verification.
5. Create every new skill as a folder containing `SKILL.md`:
   ```text
   D:\OB\skills\<skill-slug>\SKILL.md
   ```
   Use lowercase kebab-case slugs, e.g. `obsidian-skills-library`. Do not scatter skill files into `D:\.system\.pi\agent\skills`, `C:\Users\HCH\.claude\skills`, or package directories.
6. Use the Skill Creator standard body format:
   ```markdown
   ---
   name: <skill-slug>
   description: <one-line trigger condition and purpose>
   ---

   # <Human Title>

   ## When to Use
   <trigger conditions and boundaries>

   ## Procedure
   1. <concrete step>
   2. <concrete step>

   ## Pitfalls
   - <known failure mode>

   ## Verification
   1. <check that proves it works>
   ```
7. Keep descriptions trigger-oriented. The `description` should tell the agent when to load the skill, not merely summarize the file.
8. After creating or renaming a skill, update `D:\OB\skills\SKILLS_INDEX.md`:
   - add the skill under the most relevant category
   - update quick-selection rows if it is a high-priority workflow
   - do not over-edit stale historical counts unless you are doing a full reindex
9. Verify the Pi/Claude skill bridge after changes:
   ```powershell
   Get-Item 'D:\.system\.claude\skills' -Force | Format-List FullName,Attributes,LinkType,Target
   Test-Path 'D:\.system\.claude\skills\SKILLS_INDEX.md'
   Test-Path 'D:\.system\.claude\skills\<skill-slug>\SKILL.md'
   ```
   If the junction was deleted, recreate it:
   ```powershell
   New-Item -ItemType Directory -Force 'D:\.system\.claude' | Out-Null
   cmd /c mklink /J D:\.system\.claude\skills D:\OB\skills
   ```
10. Use Obsidian CLI to open the new/changed skill for human review:
    ```powershell
    & 'C:\Program Files\Obsidian\Obsidian.exe' open path='skills/<skill-slug>/SKILL.md' newtab
    ```

## Pitfalls
- Do not treat a skill name as an MCP tool name. Skills are Markdown instructions under `D:\OB\skills`; MCP tools are separate callable tools defined by MCP servers.
- Do not use `C:\Users\HCH\.claude\skills` as the source of truth. It is only a compatibility path/junction layer; the durable library is `D:\OB\skills`.
- Do not delete or overwrite package-bundled skills such as `pi-mcp-adapter\skills\mcp-scripting`; those belong to npm packages and are restored by reinstalling the package, not by editing the vault.
- The official Obsidian CLI requires the Obsidian desktop app to be running. If the CLI fails while the files exist on disk, check whether Obsidian is open before assuming the vault is corrupt.
- Obsidian CLI command-line `content=` arguments are awkward for large `SKILL.md` bodies because quoting/newlines can break in shells. For long skills, write the file directly to `D:\OB\skills\<slug>\SKILL.md`, then use `obsidian read/open/search` to verify and review.
- Avoid permanent deletes through CLI unless the user explicitly confirms. Prefer moving/renaming skill folders or keeping a backup when retiring a skill.
- Do not update `SKILLS_INDEX.md` blindly from old counts; if exact totals matter, rescan the directory first.

## Verification
1. `& 'C:\Program Files\Obsidian\Obsidian.exe' vault` reports the `OB` vault at `D:\OB`.
2. `D:\OB\skills\<skill-slug>\SKILL.md` exists and starts with valid YAML front matter containing `name` and `description`.
3. The skill has all standard sections: `## When to Use`, `## Procedure`, `## Pitfalls`, and `## Verification`.
4. `D:\OB\skills\SKILLS_INDEX.md` references the skill slug in the appropriate category.
5. `Get-Item 'D:\.system\.claude\skills' -Force` reports `LinkType : Junction` and `Target : {D:\OB\skills}`.
6. `Test-Path 'D:\.system\.claude\skills\<skill-slug>\SKILL.md'` returns `True`, proving Pi/Claude can see the Obsidian-backed skill.
7. Obsidian CLI can open or read it:
   ```powershell
   & 'C:\Program Files\Obsidian\Obsidian.exe' open path='skills/<skill-slug>/SKILL.md' newtab
   & 'C:\Program Files\Obsidian\Obsidian.exe' search query='<skill-slug>'
   ```

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
