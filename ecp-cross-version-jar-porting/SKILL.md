---
name: ecp-cross-version-jar-porting
description: Fix a "reduced build" Chainsea/ECP/aipower deployment (a Unit/module whose metadata exists in the DB but whose Java backend is missing, stubbed, or genuinely buggy in the installed vendor jar) by borrowing compiled classes from a differently-versioned complete vendor jar found elsewhere on disk. Covers the extract-scan-deploy-restart-read-stacktrace iteration loop, JSP/JS webapp-docroot shadowing, and the specific gotchas that make this riskier than a same-version class override.
triggers:
  - Unit 不存在
  - 不存在單元編碼為
  - NoClassDefFoundError chat
  - 半套 module
  - 精簡版 aipower
  - reduced build
  - 借用新版 jar
  - 跨版本 class
  - vendor 半成品
  - groups is null
  - TypeNotPresentException
---

# ECP Cross-Version Jar Porting Skill

## When to use this (vs `ecp-vendor-class-override` vs `chainsea-feature-migration`)

| Skill | Use when |
|---|---|
| `ecp-java-core` | Writing a **new** `com.chainsea.ecp.*` business module from scratch (six-layer pattern). |
| `ecp-vendor-class-override` | A **framework** class (`com.jeedsoft.quicksilver.*`) needs behavior you must hand-write, because no alternate version exists anywhere — decompile with `javap`, rewrite by hand, shadow via `WEB-INF/classes`. |
| `chainsea-feature-migration` | Moving a feature **between two live deployments** you control (DB export/import, resource copy). |
| **This skill** | A `com.chainsea.ecp.*` Unit's **DB metadata already exists** (`TsUnit`/`TsPage`/...) but the **compiled classes are missing, stub-only, or provably buggy**, and you have found a **differently-versioned vendor jar** (e.g. a newer installer package under `D:\ECP\...\Install\*.jar`) that contains a complete, working implementation. You port compiled `.class` files across product versions instead of writing source. |

The core difference from `ecp-vendor-class-override`: you are not decompiling to understand and
reproduce — you already have a working binary from another version. The job is dependency-chasing
and compatibility verification, not authorship.

## Recognizing a "reduced build" up front

Signs a deployment shipped intentionally or accidentally incomplete for a given feature:

- `TsUnit.FHomeClassName` etc. point at a `com.chainsea.ecp.*` class that **does not exist in any
  jar** under `WEB-INF/lib` (`unzip -l *.jar | grep ClassName` across the whole lib dir comes up
  empty). The DB metadata was installed; the code jar for that module was not.
- A Unit's six layers exist but only **some** (e.g. only `Model` — a bare data holder with no
  Home/Dao/Service/Action). Business logic was stripped, structure metadata was not.
- A vendor method is present but is a literal stub: `List groups = null;` followed immediately by
  `groups.isEmpty()` in the *original, unmodified* bytecode (verify with `javap -c`, not just the
  decompiler's guess — see Gotcha 6). This is the vendor's own incomplete code, not something you
  broke.
- A `TsScript`/`TsPage` row references a static resource path that has **never existed in any
  version** of the jar (check both the version installed and any newer version you can find) —
  often a typo in the vendor's own upgrade SQL (e.g. `serviceconcult` vs the real folder
  `serviceconsult`). This is dead weight, not a missing install — see Gotcha 8.

## Finding a donor jar

Look for any other Chainsea/ECP/aipower install package on the machine or reachable drives —
`Install/*.jar` under a staging directory, a second deployment's `WEB-INF/lib`, a `.7z`/`.zip`
distribution. Compare version strings in the jar filename (e.g. `aipower-module-base-7.3.12.5.jar`
vs `ecp-module-main-8.5.03.02.jar`) — these are commonly **different product line names for the same
codebase at different release points**, not unrelated products. A same-file-count directory listing
(`unzip -l jarA | grep <package> | wc -l` vs `unzip -l jarB | ...`) quickly shows whether the donor
has *more* classes for the module you need (bigger = has the real implementation; equal or fewer =
probably not your donor).

Also grab that donor package's own SQL install/upgrade script if present (e.g. `Install/ecp.sql`) —
it is the fastest way to find the *exact* `CREATE TABLE`, `TsUnit`, `TsDictionary`, `TsScript`, and
`TsToolItem` rows a module needs, instead of guessing column lists by hand.

Also check for small **versioned upgrade patches** alongside the main install SQL, e.g.
`QS-MODULE/data/sql/default/v8/8.0.6/ecp-8.0.6.01.sql` — these are the product's own incremental
migration scripts, and they're gold: a two-line `UPDATE TsUnit SET FHomeClassName=... WHERE
FId=...` in one of these tells you *exactly* which existing built-in Unit needs its class pointers
swapped (see Gotcha 10), which a full-schema diff won't surface as clearly.

## The iteration loop

This is the same loop every time, and it is faster than trying to fully static-analyze first:

1. **Extract** the target class(es) plus (on the first pass) its 5-7 sibling layers
   (`Home`/`Dao`/`DaoImpl`/`Model`/`Service`/`ServiceImpl`/`Action`/`ActionImpl`) from the donor jar
   into a scratch dir.
2. **Scan declared dependencies**: `javap -p -c <class>.class | grep -oE "com/chainsea/ecp/[a-z]+/(model/)?[A-Za-z]+"`
   across every extracted file, `sort -u`. This finds classes referenced **inside method bodies**
   only — it does **not** find everything a static initializer touches (see Gotcha 1).
3. **Cross-check** each referenced class against what's already loadable
   (`unzip -l *.jar | grep <ClassName>` across all of `WEB-INF/lib`, plus
   `WEB-INF/classes` for anything already ported this session). Anything missing needs the same
   extract treatment, recursively.
4. **Diff risky overrides before swapping.** If you're about to shadow a class that's shared with
   *already-working* vendor code (an `enum`, a widely-used `Model`), diff old vs. new with
   `javap -p` first. A **purely additive** diff (new fields/constants appended, nothing renamed or
   reordered) is safe to swap wholesale. Anything else is a smaller, riskier surgical patch, not a
   full swap — reconsider before deploying.
5. **Deploy**: copy into `WEB-INF/classes/<package/path>/`, preserving directory structure.
   Wildcard-copy (`ClassName*.class`) to catch anonymous-inner-class siblings.
6. **Restart, then grep the log for the first ERROR/`NoClassDefFoundError`/`NoSuchFieldError`/
   `NoSuchMethodError`/`TypeNotPresentException`**, not just "did it start". A clean "Server startup"
   line does not mean the ported module is error-free — many of these only surface when the actual
   feature is exercised (button click, form load), not at boot.
7. **Read the exact class+line the stack trace blames**, extract/scan/deploy only that one new
   thing, restart again. Resist the urge to bulk-port a whole neighboring subsystem speculatively —
   most of the time the next error is one or two classes, not twenty.
8. When a single stack trace pulls in **a wide, unrelated subsystem** (see Gotcha 1 for why this
   happens) and the class count starts климbing into double digits across domains that have nothing
   to do with the feature you're fixing, **stop and ask the user** whether to keep going. This is a
   real inflection point, not a false alarm — confirm scope before continuing, don't silently commit
   to porting a third of the product.

## Gotchas (in the order you'll likely hit them)

### 1. Static initializers (`<clinit>`) pull in far more than method-body scanning shows

`javap -p -c` dependency scanning (loop step 2) only sees what's referenced **inside method bodies**.
A `Home` class's `static` field initializers run the moment the class is first touched — e.g.:

```java
public class AsdTenantDataStoreHome {
    public static AsdTenantDataStore tenantRedisDataStore = new AsdTenantRedisDataStore();
    public static AsdTenantDataStore tenantMemoryDataStore = new AsdTenantMemoryDataStore();
    ...
}
```

**Both** branches construct unconditionally at class-load time, regardless of which one
`getTenantDataStore()` actually returns at runtime (`Cluster.isEnabled()` picks one, but both fields
already got built). If either constructor (or a field type referenced deep in one of those
constructors' own classes) pulls in a class you haven't ported, you get a `NoClassDefFoundError`
whose stack trace blames `SomeHome.<clinit>` — that's your signal to look at *static field
initializers*, not method bodies, in `SomeHome`'s own source (decompile it directly rather than
guessing from the caller's perspective).

This is also why a large, deeply-connected class (a stateful "engine" Service like an ASD/call-
routing/session-snapshot service) can transitively drag in a dozen unrelated subsystems just by
existing in `WEB-INF/classes` — its `<clinit>` chain, not the one method you actually need, is doing
the pulling. If the ballooning traces back to one such class's `<clinit>`, that's the moment for
Gotcha's step-8 stop-and-ask, not a longer extraction list.

### 2. Purely-additive enum/Model swaps are low-risk; check first with `javap -p` diff

```bash
javap -p OldVersion/SomeEnum.class > /tmp/old.txt
javap -p NewVersion/SomeEnum.class > /tmp/new.txt
diff /tmp/old.txt /tmp/new.txt
```

If the new version only **appends** constants/fields at the end and nothing existing was renamed or
reordered, old code compiled against the old version keeps working unmodified against the new
class file — safe to shadow wholesale (remember enum inner classes: `EnumName$1.class` ...
`EnumName$N.class`, one per constant, wildcard-copy all of them).

### 3. Schema drift: newer code often expects DB columns the current schema doesn't have

A ported class computing a hardcoded SQL string (common in older, non-generic-DAO ECP code) can
reference a column that was added in a later product version. Symptom: a raw
`java.lang.NoSuchFieldError: FSomeColumn`-shaped error is actually impossible for reflection-free
compiled SQL text — what you'll really see is an **SQL execution failure** naming the column, or
(for enum-based field-name constants) a genuine `NoSuchFieldError` against the *enum*, not the DB.
Either way: `SHOW COLUMNS`, diff against the donor's own install SQL for the same table, `ALTER
TABLE ... ADD COLUMN` with the exact type/collation from the donor SQL if it's a plain nullable
addition. Same due-diligence as Gotcha 2 — confirm the column is purely additive before adding it.

### 4. `log4j` 1.x-vs-2.x split

Older ported code sometimes imports `org.apache.log4j.Logger` (log4j 1.x API) directly, even in a
deployment that otherwise runs log4j 2.x (`log4j-api`/`log4j-core`). This throws
`NoClassDefFoundError: org/apache/log4j/Logger` at class-init. Fix by adding the official Apache
compatibility bridge — **not** a log4j-1.x jar (unpatched CVEs) — matching the log4j2 version already
present:

```
https://repo1.maven.org/maven2/org/apache/logging/log4j/log4j-1.2-api/<version>/log4j-1.2-api-<version>.jar
```

Confirm the exact `<version>` against the `log4j-api-*.jar`/`log4j-core-*.jar` already in
`WEB-INF/lib` before downloading. This is a network fetch of a third-party file — confirm with the
user first per the general external-fetch caution, even though it's a well-known official artifact.

### 5. JSP/JS can be overridden the same way classes are, via webapp docroot precedence

Vendor JSPs/JS ship inside the jar as `META-INF/resources/...` (Servlet 3.0 web-fragment resources).
The webapp's own docroot **always wins** over a jar's bundled resources at the same relative path —
so to patch a one- or two-line bug in a shipped JSP/JS without touching the framework's JS bundle
wholesale, copy the **original** file out of the jar, edit just the broken line, and place it at the
identical path directly under the webapp root:

```
<webapp>/ecp/page/chat/ChatRoomList.jsp   # shadows META-INF/resources/ecp/page/chat/ChatRoomList.jsp inside the jar
```

Clear any Jasper-compiled JSP cache under `apache-tomcat/work/Catalina/localhost/<app>/...` and
restart so the new source gets recompiled, not served from a stale `.class` in `work/`.

Example from a real session: a page's `<c:head import="List,Tree,Resizer">` never imported
`ListView`, but the page's Action (ported from a newer version) stopped explicitly setting
`multiRow=false` in its client data — the client-side default flipped to `true`, which needs
`Jui.option.ListView`, which was never loaded, producing `Cannot read properties of undefined
(reading 'create')`. Fixed by editing just the `import=` attribute to add `ListView`, not by
patching the Action's Java to re-add the missing default.

### 6. The vendor's own code — Java or JS — can have a genuine, shipped bug; don't assume you broke it

Decompilers can occasionally mis-render bytecode, but when in doubt read the raw disassembly, not
the decompiler's guess:

```bash
javap -p -c -l SomeClass.class | sed -n '/methodName/,/^$/p'
```

If you see `aconst_null` / `astore N` immediately followed by `aload N` and a method call on it —
that's a literal `Type x = null; x.someMethod();` **in the original, unmodified vendor bytecode**.
This is the vendor's own unfinished/broken code, not a decompiler artifact and not something you
introduced. Same applies to front-end JS: a callback can assume a response shape
(`ret.result.someField`) that the actual code path never produces, because the *real* production
flow uses a **different, custom-bound button** than the one you're clicking — see Gotcha 7.

### 7. Find the *actual* bound handler for a button before assuming a generic framework action applies

A list/selection page's **visible "確定/OK" toolbar item is not always the generic framework
confirm** (`EntityList.doSelectListConfirm`, which just closes the dialog with the raw selected
pairs and calls **no backend API at all**). Query the page's own metadata for a custom-bound
`TsToolItem`:

```sql
SELECT FCode, FName, FDefaultEventHandler FROM TsToolItem WHERE FPageId='<page id>';
SELECT FUrl FROM TsScript WHERE FPageId='<page id>';
```

If a row like `FDefaultEventHandler='SomePageNamespace.someRealSubmitMethod();'` exists alongside a
`TsScript` loading a page-specific `.js` file, **that** function is the real save/confirm path — it
typically builds the correct request payload and calls the actual backend API, whereas the generic
`doSelectListConfirm` only returns raw selection data to the caller's JS callback. Calling the wrong
one **looks like it worked** (dialog closes, no console error) but silently skips the entire backend
side-effect (e.g. no "check-in" state is ever written). Verify success by checking the *effect*
(re-query a status API, not just "no error was thrown").

### 8. Distinguish "genuinely missing" from "dead metadata that never worked in any version"

Before treating a 404'd script/page reference as something to port, check whether it existed in
**any** version of the vendor jar you can find, including the donor. If a `TsScript`/`TsPage` row
references a path with an obvious typo (compare against a similarly-named folder that *does* exist,
e.g. resource images under `serviceconsult/` vs a script path under `serviceconcult/`) and neither
version ever shipped that file, it's leftover dead metadata from the vendor's own upgrade SQL, not a
missing install. Fix: back up the row (`mysqldump --where`), then `DELETE` it — don't try to
reconstruct a phantom file. See `ecp-schema`'s safe-mutation workflow for the backup step.

### 9. Runtime state (session/in-memory/Redis snapshots) is not database state

Some "is this feature working" checks live entirely in an in-process cache or Redis, not any SQL
table — e.g. a call-center "checked-in agent" set backing a ready/not-ready toggle. Don't spend time
searching for a persistent table to verify these; instead call the read-side API directly
(`Utility.invoke` from the browser console via automation, or trigger the UI action) and check the
**response JSON**, not a database row, to confirm state actually changed.

### 10. A `Home` subclassing a *framework* base class needs an existing built-in `TsUnit` UPDATEd, not a new row INSERTed

Not every `NoClassDefFoundError`/`getService()==null` for a ported class means "insert a fresh
`TsUnit` row." If the ported `Home` extends a `com.jeedsoft.quicksilver.*` base (e.g. `EcpTenantHome
extends TenantHome`), it's a **product-level override of an already-existing built-in Unit**
(`Qs.Tenant` in this case), not a new module. Check whether the donor's own versioned upgrade SQL
(see the "Finding a donor jar" section) has an `UPDATE TsUnit SET FHomeClassName=...,
FDaoClassName=..., FServiceClassName=... WHERE FId='<fixed id>'` — if so, that fixed `FId` already
exists locally pointing at the stock framework classes; swap only those three columns to the Ecp
subclass names. Inventing a brand-new `TsUnit` row instead will not wire up correctly, because the
framework's own `TenantListener`/module bootstrap resolves this Unit by its well-known `FId`, not by
code lookup.

### 11. A `TsToolItem` can already be correctly configured for a dictionary-driven dropdown while the dictionary itself is missing

`TsToolItem.FType='ComboButton'` + `FSubItemSource='Dictionary'` + `FDictionaryId=<id>` is the
complete, correct metadata for a toolbar split-button whose sub-items come from a `TsDictionary` —
don't assume a missing dropdown means broken JS or missing `TsToolSubItem` rows. Check first whether
the referenced `TsDictionary`/`TsDictionaryItem` rows actually exist:

```sql
SELECT * FROM TsDictionary WHERE FId = (SELECT FDictionaryId FROM TsToolItem WHERE FCode='...');
```

Zero rows means the dictionary reference is a dangling pointer — port the `TsDictionary` +
`TsDictionaryItem` rows from the donor SQL (search the donor SQL for the dictionary's own `FId`, not
just its name, since names can collide). Once populated, the ComboButton's item list resolves
automatically from existing framework code — no JS change needed.

### 12. Dictionary items resolved into toolbar/combobox metadata can be baked in at first render and need a restart to refresh

A `TsDictionaryItem` INSERT is plain business data and normally takes effect immediately on the next
request — but if a `ComboButton`'s `items` array was already resolved to `[]` on an **earlier**
render of that toolbar (before the dictionary rows existed) and the client-side widget's `_args`
still shows the stale empty array after a fresh page load, the *page* is fresh but the underlying
resolution may have been server-cached or the browser tab restored pre-fix session state. If a
brand-new browser tab plus DB fix still shows an empty dropdown, restart Tomcat once before assuming
the JS is broken — dictionary-backed toolbar item lists observed this session only picked up new
rows after a restart, not on the next request.

### 13. A "broken" UI control can be genuine vendor code correctly honoring a business config flag that's just unset in your data

Before concluding a widget (checkbox/radio/combo embedded in a list column) is non-functional,
decompile-check whether the vendor's own JS explicitly disables it based on a per-row server field:

```js
if (!groupRow.FCustomSkillLevelEnabled) {
    weightBoxs[groupId].setDisabled(true);
}
```

A widget stuck disabled for every real mouse click, but that responds fine when you call its
`setValue()` API directly via `browser_evaluate`, is the signature of this — the JS API bypasses the
`disabled` gate entirely, which can mislead you into thinking the *feature* works when only your
scripted bypass does. Trace the boolean back to its source column (`SHOW COLUMNS FROM
TcSomeTable LIKE '%Enabled%'`) and check the actual row's value before touching any code; this is
business seed data, not a porting defect, and fixing it is a plain `UPDATE`, not a class port.

### 14. Force-killing Tomcat via its `java.exe` PID leaves the embedded `mariadbd.exe` child running and holding file locks

`Stop-Process -Force` on the Tomcat `java.exe` PID does not kill `mariadbd.exe` (MariaDB4j spawns it
as a separate child process; Windows does not cascade-kill children by default). The orphaned
`mariadbd.exe` keeps `ibdata1` open, so the *next* `server.bat` start fails with `InnoDB: The data
file './ibdata1' must be writable` → `Failed to grow the connection pool` →
`One or more listeners failed to start` / `Context [/aipower] startup failed due to previous
errors` — a full webapp-context boot failure, not a Unit-specific error. Before restarting, always
also stop the `mariadbd.exe` process:

```powershell
$mariadbd = Get-CimInstance Win32_Process -Filter "Name='mariadbd.exe'" | Select-Object -ExpandProperty ProcessId
if ($mariadbd) { Stop-Process -Id $mariadbd -Force }
```

If you skip this and hit the failure anyway, the fix is the same: kill the leftover `mariadbd.exe`
(check `Get-CimInstance Win32_Process -Filter "Name='mariadbd.exe'"`, it'll be there with an old
start time), *then* restart — restarting again without killing it first just repeats the failure.

## Session pointer

A concrete worked example spanning most of these gotchas (Ecp.ChatHistory / Ecp.ChatWorkGroup /
Ecp.SkillSetup / Ecp.ProblemType / the ChatAsd engine, ported from a `7.3.12.5` deployment to
classes borrowed out of an `8.5.03.02` installer jar) lives in this project's session history —
search for "ChatRoomExcelFiled" or "setCheckinWorkGroups" if you need the blow-by-blow.

---

## Conformance Addendum

## When to Use
Fix a "reduced build" Chainsea/ECP/aipower deployment (a Unit/module whose metadata exists in the DB but whose Java backend is missing, stubbed, or genuinely buggy in the installed vendor jar) by borrowing compiled classes from a differently-versioned complete vendor jar found elsewhere on disk. Covers the extract-scan-deploy-restart-read-stacktrace iteration loop, JSP/JS webapp-docroot shadowing, and the specific gotchas that make this riskier than a same-version class override.

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
