---
name: ecp-knowledge
description: Manage the cbm-lite knowledge base in aipower — add/search entries via UI or direct DB query, understand TcProcessKnowledge schema
triggers:
  - ecp knowledge
  - knowledge base
  - 知識庫
  - TcProcessKnowledge
argument-hint: "[add|search|check] [keyword]"
---

# ecp-knowledge Skill

## Purpose

The cbm-lite knowledge base is stored in `TcProcessKnowledge` in the aipower MariaDB4j database.  
Users add/edit knowledge through the aipower UI at **辦公自動化 → 知識庫** (list page).

## UI Path

```
aipower → 辦公自動化 → 知識庫
```
- **名稱** = question/topic title (used for search matching)
- **內容** (rich text) = the answer returned to LINE / API callers

After saving, status must be **Published** for the server to return it.

## Key DB Table

```sql
-- Check recent entries
SELECT FId, FName, FStatus, LEFT(FContent,100) FROM TcProcessKnowledge ORDER BY FCreateTime DESC LIMIT 10;
```

Connect with:
```powershell
$mysql = "C:\com\chainsea\apache-tomcat\temp\MariaDB4j\base\bin\mysql.exe"
# Get current MariaDB port first:
(Get-WmiObject Win32_Process | Where-Object { $_.Name -like "*mariad*" }).CommandLine | Select-String "--port=\d+"

& $mysql -u root -h 127.0.0.1 -P <PORT> default -e "SELECT FName, FStatus FROM TcProcessKnowledge ORDER BY FUpdateTime DESC LIMIT 5;"
```

## Search Behavior

The server searches `FName` and `FContent` (HTML stripped).

**Full match first:** exact substring anywhere in name/content.  
**CJK sliding window fallback:** for long Chinese questions with no space delimiters, tries progressively shorter substrings (min 2 chars) until a match is found.

Example: "上班時間是幾點" → tries "上班時間", "班時間", "上班", "時間" etc. until KB entry containing "時間" is found.

## Test Search via API

```powershell
$body = '{"question":"YOUR QUESTION","topN":3}'
Invoke-RestMethod -Uri "http://127.0.0.1:12621/cbm/ask" -Method POST -Body $body -ContentType "application/json" | ConvertTo-Json -Depth 3
```

Response `source` values:
- `knowledge` — answered from KB
- `llm` — KB had no match, used LLM fallback
- `none` — no answer found

## Gotchas

- **FStatus must be `Published`** — Draft entries are skipped by the search query.
- **HTML in FContent** — Rich text editor saves HTML; the server strips tags (`stripHtml()`) before matching and returning answers.
- **MariaDB port changes on every Tomcat restart** — If search returns 0 results, check the port (see `cbm-lite` skill).
- **Avoid overbroad keyword rules** — Keywords in `cbm-lite-replies.conf` intercept messages BEFORE KB search. Remove any keyword that is a common substring (e.g., `時間`).

---

## Conformance Addendum

## When to Use
Manage the cbm-lite knowledge base in aipower — add/search entries via UI or direct DB query, understand TcProcessKnowledge schema

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
