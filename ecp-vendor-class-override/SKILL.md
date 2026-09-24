---
name: ecp-vendor-class-override
description: Override a compiled Quicksilver/Jocket vendor framework class (not a chainsea business module) by decompiling with javap, reproducing its exact behavior in new source, and shadowing the jar version via WEB-INF/classes. Worked example included - real-time online-user disconnect detection (InnerJocketEndpoint).
triggers:
  - jocket
  - InnerJocketEndpoint
  - quicksilver-module-main
  - override vendor class
  - WEB-INF/classes 覆蓋
  - 框架類別修改
  - real-time online detection
argument-hint: "[fully-qualified class name to override]"
---

# ECP Vendor Class Override Skill

## When to use this (vs `ecp-java-core`)

`ecp-java-core` governs **new business modules** under `com.chainsea.ecp.*` (Model/Dao/Service/Action/Api/Home,
six-layer pattern, source lives in `tool/src`, deployed as `com/chainsea/...` into `WEB-INF/classes`).

Use **this** skill instead when the change touches **vendor framework code** —
`com.jeedsoft.quicksilver.*` (quicksilver-module-main) or `com.jeedsoft.jocket.*` (jeedsoft-jocket) —
because there is no source to extend, no six-layer contract, and the class is a singleton
framework component wired up by annotations/reflection, not `Home`/`Registry` proxies.

## The technique

Tomcat's webapp classloader resolves `WEB-INF/classes` **before** `WEB-INF/lib/*.jar`. Dropping a
`.class` with the exact same fully-qualified name into `WEB-INF/classes` silently shadows the jar's
version — no jar editing, no bytecode patching (contrast with the `mariaDB4j-core` install() patch
in `ecp-server-startup`, which patches bytes *inside* the jar because there's no classloader precedence trick
available for a library jar consumed outside a webapp).

Since there's no `.java` source for vendor classes, you must **decompile-by-reading** with `javap`
(no external decompiler needed — bytecode is usually simple enough to read directly) and
**reproduce the original method bodies verbatim** for anything you're not changing, or you will
silently drop behavior other code depends on.

### Step 1 — find the class and read it with javap

```powershell
$JAVAC = "C:\Lab\chainsea\jdk\bin\javac.exe"   # adjust root per deployment (C:\com\chainsea, C:\Lab\chainsea, ...)
$JAVAP = "C:\Lab\chainsea\jdk\bin\javap.exe"
$LIB = "C:\Lab\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib"
```

```bash
cd "$LIB"
unzip -l quicksilver-module-main-7.2.2.jar | grep -i <keyword>   # locate the class
TMPD=/tmp/vendor_extract   # or the session scratchpad dir
mkdir -p "$TMPD"
unzip -o -q quicksilver-module-main-7.2.2.jar -d "$TMPD"
"$JAVAP" -p -c -classpath "$TMPD" com.jeedsoft.quicksilver.some.package.SomeClass
```

- `-p` shows private members (you need to see everything the class does).
- `-c` disassembles method bodies — read the `invokestatic`/`invokevirtual`/`invokeinterface` call
  sequence like a very verbose transcript of the original source; it reads more easily than it looks.
- `-v` on a *fresh, separate* extraction of the ORIGINAL (unmodified) class additionally prints
  `RuntimeVisibleAnnotations` — **do this even if the class looks like a plain POJO**. See the
  annotation gotcha below; skipping this step is the single most likely way to crash startup.

### Step 2 — trace supporting types the same way

Method calls in the disassembly reveal collaborator classes (Home/Service/Dao/Model). Extract and
`javap -p` each one to learn field/method signatures (e.g. `OnlineUserModel.getSocketId()`,
`OnlineUserDao.updateSocketId(DaoContext, String, String)`) before writing new code — guessing
signatures wastes a compile-deploy-restart cycle per guess.

### Step 3 — write the replacement source

Place at the project's fixed dev path (`tool/src/<SimpleClassName>.java`, flat file, correct
`package` declaration inside — see `ecp-java-core` for the exact path convention). Reproduce
**every** method from the original, not just the one you're changing — there is no `super` to fall
back on for a class that only `implements` an interface with no base class to extend.

**Gotcha — annotations are invisible in a plain `javap -p` listing and get silently lost:**
A framework class picked up by reflection-based deployers (e.g. Jocket's `JocketDeployer`) is
usually marked with an annotation carrying config the deployer needs (e.g.
`@JocketServerEndpoint("/jocket/inner")` telling Jocket which URL path this endpoint answers).
`javap -p -c` alone does **not** show annotations — you must add `-v` (or specifically grep the
constant pool for `RuntimeVisibleAnnotations`) on the *original* class to see them. Omitting the
annotation compiles fine, deploys fine, but explodes at Tomcat startup with something like:

```
java.lang.NullPointerException: Cannot invoke "...Annotation.value()" because "annotation" is null
    at com.jeedsoft.jocket.endpoint.JocketEndpointConfig.<init>(...)
    at com.jeedsoft.jocket.endpoint.JocketDeployer.deploy(...)
```

Fix: re-add the exact annotation with its exact value, found via `-v`.

### Step 4 — compile

```bash
"$JAVAC" -encoding UTF-8 -cp "$LIB/*" -d "C:/Lab/chainsea/tool/classes" "C:/Lab/chainsea/tool/src/SomeClass.java"
```
Zero output = success. Any collaborator type referenced must resolve via the `$LIB/*` classpath
(all vendor + business jars); if `javac` can't find a type, it's usually in a jar you didn't
expect (check with `unzip -l *.jar | grep ClassName` across the whole `WEB-INF/lib`).

### Step 5 — deploy (shadow the jar)

```bash
DEST="C:/Lab/chainsea/apache-tomcat/webapps/aipower/WEB-INF/classes/com/jeedsoft/quicksilver/some/package"
mkdir -p "$DEST"
cp "C:/Lab/chainsea/tool/classes/com/jeedsoft/quicksilver/some/package/SomeClass"*.class "$DEST/"
```

**Gotcha — anonymous inner classes compile to separate `.class` files and are easy to leave
behind:** any `new Xyz() { ... }` (anonymous `ThreadFactory`, `Runnable`, listener, etc.) or lambda
capturing state in a way `javac` needs a synthetic class for compiles to `SomeClass$1.class`,
`SomeClass$2.class`, etc. — **not** into `SomeClass.class`. Copying only `SomeClass.class` produces
a `NoClassDefFoundError: SomeClass$1` at Tomcat startup the moment the class is loaded (its static
initializer or field init references the missing inner class). Fix: glob-copy
(`SomeClass*.class`), and `ls` the source `tool/classes/.../` directory first to see exactly how
many files one `.java` produced before copying.

### Step 6 — restart and verify startup succeeded before testing behavior

Full restart procedure is in `ecp-server-startup`/project `CLAUDE.md`
(`Stop-Process -Name mariadbd, java, redis-server -Force` then `server.bat`). Check the log for:

```
Quicksilver startup in <N> ms --------
Server startup in <N> ms
```

**not** `!!!!!!!! Quicksilver startup failed !!!!!!!!`. A startup failure here takes the whole
webapp down for every user — always confirm success (and ideally `Invoke-WebRequest` 200) before
declaring done, and always get explicit user confirmation before restarting a shared instance
(restart drops every active session).

### Step 7 — prove the behavior, don't just assume the deploy worked

Query the DB field/state your patch touches directly, before and after triggering the condition.
`javap`-level confidence that the code is *correct* is not the same as evidence that the deployed
class is the one actually running (classloader precedence mistakes, stale `WEB-INF/classes` from a
previous attempt, wrong jar version extracted, etc. are all real failure modes).

## Worked example: real-time online-user disconnect detection

### Problem

`線上使用者` (online users) list only clears an entry when the passive `QsSessionTimeout` /
`QsCrowdedSessionTimeout` sweep runs (`OnlineUserServiceImpl.checkSessions`, 30 min by default —
see `ecp-schema`'s System Parameter section). A browser that's force-closed (not a clean tab close
that fires `beforeunload`) sits in the list as "online" for up to 30 minutes even though the Jocket
push connection died within seconds.

### Root cause found via javap

`com.jeedsoft.quicksilver.application.jocket.inner.InnerJocketEndpoint` (in
`quicksilver-module-main-7.2.2.jar`) implements `com.jeedsoft.jocket.endpoint.JocketEndpoint`.
`onOpen()` correctly writes the live Jocket connection id into `TsOnlineUser.FSocketId`
(`OnlineUserDao.updateSocketId(daoContext, httpSessionId, jocketSessionId)`, backed by
`update TsOnlineUser set FSocketId = ? where FSessionId = ?`). **`onClose()` only logs** — it never
clears `FSocketId` when the connection actually dies, so the field is a write-once, stale-forever
marker instead of a live-connection indicator.

### Fix

New `com.jeedsoft.quicksilver.application.jocket.inner.InnerJocketEndpoint` (shadows the jar's
version). `onOpen`/`onMessage` reproduce the original bytecode exactly; `onClose` additionally
clears `FSocketId` — but **only if it still matches the closing connection's own id**, to avoid a
race where a fast reconnect already wrote a fresher socket id before the old connection's close
event is processed:

```java
package com.jeedsoft.quicksilver.application.jocket.inner;

import java.util.UUID;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import javax.servlet.http.HttpSession;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.jeedsoft.jocket.connection.JocketCloseReason;
import com.jeedsoft.jocket.connection.JocketSession;
import com.jeedsoft.jocket.endpoint.JocketEndpoint;
import com.jeedsoft.jocket.endpoint.JocketServerEndpoint;
import com.jeedsoft.quicksilver.account.model.Identity;
import com.jeedsoft.quicksilver.base.type.DaoContext;
import com.jeedsoft.quicksilver.base.type.ServiceContext;
import com.jeedsoft.quicksilver.user.OnlineUserHome;
import com.jeedsoft.quicksilver.user.model.OnlineUserModel;

@JocketServerEndpoint("/jocket/inner")   // REQUIRED - see annotation gotcha above
public class InnerJocketEndpoint implements JocketEndpoint
{
    private static final Logger logger = LoggerFactory.getLogger(InnerJocketEndpoint.class);
    private static final long DISPLAY_DEBOUNCE_SECONDS = 8;   // cosmetic: clear FSocketId
    private static final long KICK_DEBOUNCE_SECONDS = 90;     // real logout: releases held work

    private static final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(
            new ThreadFactory() {
                @Override public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "InnerJocket-disconnect-debounce");
                    t.setDaemon(true);
                    return t;
                }
            });

    @Override
    public void onOpen(JocketSession session, HttpSession httpSession)
    {
        if (httpSession == null) { session.close(4900, "HTTP session is null."); return; }
        OnlineUserModel onlineUser = OnlineUserHome.getService().getItem(httpSession);
        if (onlineUser == null) { session.close(4900, "Invalid online user."); return; }
        Identity identity = onlineUser.getIdentity();
        session.setUserId(InnerJocket.joinId(onlineUser.getTenantId(), identity.getEntityId()));
        session.setOnlineUserId(onlineUser.getId().toString());
        String httpSessionId = httpSession.getId();
        InnerJocketSessionProperty.setHttpSessionId(session, httpSessionId);
        OnlineUserHome.getDao().updateSocketId(DaoContext.getDefaultInstance(), httpSessionId, session.getId());
    }

    @Override
    public void onClose(JocketSession session, JocketCloseReason reason)
    {
        String httpSessionId = InnerJocketSessionProperty.getHttpSessionId(session);
        if (httpSessionId == null) return;
        String closedSocketId = session.getId();
        scheduler.schedule(() -> clearSocketIfStillDisconnected(httpSessionId, closedSocketId),
                DISPLAY_DEBOUNCE_SECONDS, TimeUnit.SECONDS);
        scheduler.schedule(() -> kickOutIfStillDisconnected(httpSessionId, closedSocketId),
                KICK_DEBOUNCE_SECONDS, TimeUnit.SECONDS);
    }

    // FSocketId is either unchanged since this close (never reconnected) or already null
    // (cleared by our own display debounce, or never set) - both mean no reconnect has
    // claimed this row since. Any OTHER value means a newer connection owns it - back off.
    private static boolean isStillDisconnected(OnlineUserModel current, String closedSocketId)
    {
        if (current == null) return false; // gone some other way already (timeout, manual kick...)
        String socketId = current.getSocketId();
        return socketId == null || closedSocketId.equals(socketId);
    }

    private static void clearSocketIfStillDisconnected(String httpSessionId, String closedSocketId)
    {
        OnlineUserModel current = OnlineUserHome.getService().getItem(httpSessionId);
        if (isStillDisconnected(current, closedSocketId)) {
            OnlineUserHome.getDao().updateSocketId(DaoContext.getDefaultInstance(), httpSessionId, null);
        }
    }

    private static void kickOutIfStillDisconnected(String httpSessionId, String closedSocketId)
    {
        OnlineUserModel current = OnlineUserHome.getService().getItem(httpSessionId);
        if (isStillDisconnected(current, closedSocketId)) {
            try {
                ServiceContext ctx = ServiceContext.getDefaultInstance().notCheckAccess();
                OnlineUserHome.getService().kickOut(ctx, new UUID[]{ current.getId() });
            }
            catch (Exception e) {
                logger.error("[InnerJocket] Failed to kick out httpSessionId=" + httpSessionId, e);
            }
        }
    }

    @Override
    public void onMessage(JocketSession session, String name, String data) { }
}
```

### Why two separate debounces, not one

`DISPLAY_DEBOUNCE_SECONDS` (8s) only touches a display field - worst case for a false positive is
a flicker in a query-filtered list, so it can be short and reactive. `KICK_DEBOUNCE_SECONDS` (90s)
does a **real logout** via the same path as the manual 踢出 button - a false positive here actually
force-logs-out a working user, which is strictly worse than the original "動不動被登出" complaint
this whole investigation started from. Keep these on separate timers with separate constants; do
not collapse them into one delay. 90s was chosen as "long enough that tab-backgrounding, a WiFi
blip, or one of our own Tomcat restarts survives it, short enough to still beat the 30-minute
QsSessionTimeout sweep by over an order of magnitude."

### Why real kick-out at all, not just hiding from the list

The display-only version (clearing `FSocketId`) solves "the online list lies," but not the actual
business problem: whatever the disconnected account was holding — an assigned chat, a task, an
edit lock — stays locked to that dead session until the 30-minute sweep finally runs `checkSessions`
and removes the row, so nobody else can pick up the work in the meantime. `OnlineUserService.
kickOut(ServiceContext, UUID[])` walks the same code path as an admin manually selecting the row
and clicking 踢出: it deletes the `TsOnlineUser` row, logs a `TsLoginLog` row with `FAction=KickOut`,
and pushes `Qs.Session.Invalidate` over Jocket. Whatever release-on-logout logic the app already has
for a normal logout fires the same way — no new business logic needed, just triggering the existing
logout path proactively instead of waiting for the user or the timeout sweep.

### `ServiceContext` for a call with no HTTP request behind it

`kickOut` calls `Privilege.checkGlobalPrivilege(ctx, ...)` internally, which requires a manage
privilege — normally satisfied by whichever admin's request context is calling it. A background
scheduled task has no such request/no such admin. `Privilege.checkGlobalPrivilege`'s first bytecode
instruction is `ctx.isCheckAccess()` → if false, `return` immediately, skipping the whole privilege
lookup (verified via `javap -c` before relying on it — don't assume, check). So
`ServiceContext.getDefaultInstance().notCheckAccess()` is exactly the escape hatch needed: same
idiom as `DaoContext.getDefaultInstance()` used throughout `onOpen`, just for the service layer.

### Measured results (2026-07-02)

```
T+0s    Chrome process force-killed (not a clean tab close)
T+9s    TsOnlineUser.FSocketId already NULL          (pre-debounce version, superseded)
```
Far under the theoretical worst case (`pingInterval` 25s + `pingTimeout` 20s + 5s ≈ 50s from
`JocketService`'s hardcoded defaults) because an abrupt TCP close is detected by the socket read
failing immediately, not by waiting for a missed ping. With the dual-debounce version deployed:
expect `DISPLAY_DEBOUNCE_SECONDS` added for the list to update, `KICK_DEBOUNCE_SECONDS` added for
the actual logout (~90-140s total from a genuine disconnect to `TsLoginLog` showing `KickOut`), and
*no* action at all for any reconnect landing inside either window.

### Surfacing it in the UI without touching QsSessionTimeout

Do **not** shorten `QsSessionTimeout`/`QsCrowdedSessionTimeout` to make the list "feel" real-time —
that reintroduces aggressive logout of genuinely-idle-but-still-connected users (see `ecp-schema`'s
Session parameter section for that whole saga). Instead add a query scheme on the existing
`線上使用者` screen (`Qs.OnlineUser` unit) filtering the already-exposed field `雙向連線 ID`
(`FSocketId`) with operator **有資料** (IS NOT NULL/has data). This is purely a display filter; the
underlying HTTP session / login timeout logic is completely untouched.

```sql
-- verify a query scheme after creating it through the UI (UI query builder is safer than
-- hand-crafting TsQuerySchema/TsQueryCondition rows - format is nontrivial)
SELECT FId, FName, FUnitId, HEX(FPublic) AS is_public FROM TsQuerySchema WHERE FName='真正在線(有心跳)';
-- flip visibility to public if the UI checkbox widget doesn't respond to automation clicks
-- (it's a custom div-based widget, not a native <input type=checkbox> - browser_evaluate querySelector
-- for input[type=checkbox] finds nothing)
UPDATE TsQuerySchema SET FPublic=1 WHERE FId='<id from above>';
```

### Making it the screen's *default* view (not just selectable)

`TsQuerySchema.FGlobalAutoQuery` looks like the obvious knob ("global auto query") but **setting it
to `1` had no effect** — the screen still opened with the query dropdown blank. The field that
actually controls which scheme a page opens with is `TsPage.FQuerySchemaId` on the specific page
record (e.g. `Qs.OnlineUser.List`, found the same way as any page — see `ecp-schema`'s Menu→Page→Unit
trace):

```sql
SELECT FId, FCode, FQuerySchemaId FROM TsPage WHERE FUnitId='<unit id>' AND FCode LIKE '%.List';
UPDATE TsPage SET FQuerySchemaId='<query scheme FId>' WHERE FId='<page FId>';
```

**This is Registry-cached exactly like `TsField`** (see `ecp-schema`'s note that `TsField` edits
need a restart to reload) — a page reload, a brand new tab, even a fresh browser process are *all*
insufficient; only a Tomcat restart picks it up. Don't waste cycles debugging "why doesn't my fresh
tab see the new default" — it's not a caching layer you can bust from the client side.

## Reusable pattern for other "passive timeout should be an active event" gaps

The same shape — a framework class has the right event hook (`onClose`, a `*RemoveListener`
interface, etc.) but nothing reads/writes the field you need — shows up wherever ECP tracks
liveness with a "last seen" timestamp instead of a real disconnect signal. Before assuming you need
to shorten a timeout parameter, `javap` the relevant lifecycle class first; there is often an unused
hook already carrying the exact event you want.

---

## Conformance Addendum

## When to Use
Override a compiled Quicksilver/Jocket vendor framework class (not a chainsea business module) by decompiling with javap, reproducing its exact behavior in new source, and shadowing the jar version via WEB-INF/classes. Worked example included - real-time online-user disconnect detection (InnerJocketEndpoint).

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
