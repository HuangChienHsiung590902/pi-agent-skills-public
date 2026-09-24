---
name: ecp-menu
description: Add or modify aipower left-sidebar menu items directly via TsMenu DB — rename, add shortcuts, wire to existing pages
triggers:
  - ecp menu
  - aipower menu
  - TsMenu
  - 選單
argument-hint: "[add|rename] <menu-name>"
---

# ecp-menu Skill

## Purpose

aipower menu structure is stored in `TsMenu`. You can add new menu shortcuts to existing pages (e.g., add a "知識庫" entry that points to ProcessKnowledge list) without writing any Java code.

## Menu Hierarchy

```
Level 1 — sidebar icon group  (e.g., 辦公自動化: 00000000-0000-0000-0008-020000000010)
  Level 2 — section header
    Level 3 — top-level item  (FTreeSerial: 002.001.XXX)
      Level 4 — sub-item
        Level 5 — leaf item
```

## Find Existing Entry

```powershell
$mysql = "C:\com\chainsea\apache-tomcat\temp\MariaDB4j\base\bin\mysql.exe"
& $mysql -u root -h 127.0.0.1 -P <PORT> default -e "
SELECT FId, FParentId, FName, FTreeLevel, FIndex, FPageId, FIcon
FROM TsMenu WHERE FName LIKE '%標準作業%';"
```

## Add a New Shortcut Menu Entry

Pattern: copy an existing entry's `FPageId`, give it a new name and parent.

```powershell
$newId = [guid]::NewGuid().ToString()
$sql = @"
INSERT INTO TsMenu (
  FId, FParentId, FIndex, FTreeLevel, FTreeSerial,
  FName, FIcon, FType, FPageId,
  FEnabled, FBuiltin, FHideInMainMenu, FIsSystemLevel, FReplaceByChildren
) VALUES (
  '$newId',
  '<PARENT_FId>',
  <INDEX>, <LEVEL>, '<TREE_SERIAL>',
  '<DISPLAY_NAME>',
  'ecp/image/unit/Literature.gif',
  'InternalPage',
  '<TARGET_PAGE_FId>',
  1, 0, 0, 0, 0
);
-- Copy role permissions from source menu entry
INSERT INTO TsRoleMenu (FRoleId, FMenuId)
  SELECT FRoleId, '$newId' FROM TsRoleMenu WHERE FMenuId = '<SOURCE_MENU_FId>';
"@
& $mysql -u root -h 127.0.0.1 -P <PORT> default -e $sql
```

Then **F5** in the browser — new menu appears immediately, no Tomcat restart needed.

## Known Mappings (this install)

| Menu | FId | FPageId |
|------|-----|---------|
| 辦公自動化 | `00000000-0000-0000-0008-020000000010` | — |
| 標準作業流程 | `c5715124-7792-42e6-864e-e84c370f0a6d` | `f8c786e4-644b-4b68-be81-30eda38cd34e` |
| 知識庫 (added) | `bfdd35ad-d652-4683-a61a-7e15e71b5e84` | same as above |

## Available Icons

```
quicksilver/image/16/Home.gif
ecp/image/unit/Literature.gif      ← good for knowledge base
ecp/image/unit/Contact.gif
quicksilver/image/unit/SystemMessage.gif
ecp/image/unit/Activity.gif
```

## FTreeSerial Pattern

For level-3 items under 辦公自動化: `002.001.XXX` (e.g., `002.001.007` for index 7).

## Role Permissions

`TsRoleMenu(FRoleId, FMenuId)` — copy from source entry to grant same access.  
Default roles in this install:
- `00000000-0000-0000-1004-000000000001`
- `00000000-0000-0000-1004-100000000001`

---

## Conformance Addendum

## When to Use
Add or modify aipower left-sidebar menu items directly via TsMenu DB — rename, add shortcuts, wire to existing pages

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
