# pi-agent-skills (public mirror)

Public distribution mirror of the local `D:\OB\skills` skill library, intended for
skill managers (e.g. CC Switch) that download repositories anonymously.

Contents:

- `<skill-id>/SKILL.md` for every skill in the library (243 skills)
- `SKILLS_INDEX.md` and `_consolidation/` machine-readable indexes

Deliberately excluded from this mirror:

- executable `scripts/`, `references/`, `assets/` side files
- internal Java/SQL source, deployment configs, and credentials
- any root `SKILL.md` (so skill managers enumerate each skill folder separately)

The canonical, full library (with scripts and references) lives on the private
repository `HuangChienHsiung590902/pi-agent-skills` and on the local machine at
`D:\OB\skills`.

All passwords, tokens and host credentials found in skill text have been
replaced with `<PLACEHOLDER>` markers in this mirror.
