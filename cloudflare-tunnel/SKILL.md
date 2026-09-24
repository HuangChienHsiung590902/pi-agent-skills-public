---
name: cloudflare-tunnel
description: Manage the multi-machine Cloudflare Tunnel "hch" (cloudflared) — connector status across WIN11-1/SRV/this laptop, config, restart service, add hostname routes, and the Windows service-install gotchas
triggers:
  - cloudflare tunnel
  - cloudflared
  - tunnel
argument-hint: "[status|restart|add-route] [hostname]"
---

# cloudflare-tunnel Skill

## Purpose

Expose services to the internet via Cloudflare Tunnel without opening firewall ports.
Tunnel name: **hch**. Tunnel ID: `fca39237-d676-470f-ad9b-4fe870d94026`. Zone: `james-huang.org`.

## Current architecture (as of 2026-08-15) — multiple connectors, not one machine

This tunnel is **not** single-machine anymore. It's an HA setup with cloudflared running as a Windows service on two separate boxes, both authenticating as connectors for the same tunnel ID:

| Machine | Role | `.cloudflared` folder | Ingress points services at |
|---|---|---|---|
| **WIN11-1** (`192.168.2.201`, home-lab Windows 11 client VM) | **Primary** — this is where the actual origin service (aipower Tomcat) runs | `C:\Users\administrator\.cloudflared\` (note: lowercase `administrator`, **not** `Administrator.LAB` — that was an open question, now confirmed) | `127.0.0.1:22821` |
| **SRV** (`192.168.2.200`, home-lab AD Domain Controller VM) | **Secondary / HA** — pure connector, forwards cross-machine over the same `192.168.2.x` subnet | `C:\Users\Administrator\.cloudflared\` | `192.168.2.201:22821` (i.e. WIN11-1's LAN IP — **not** `127.0.0.1`, SRV doesn't run the service itself) |
| **This laptop** (Gram, `C:\Users\HCH\.cloudflared\`) | **Historically primary, currently broken/out of scope** — see "Deep Freeze wipes this machine" below | n/a right now |

> To SSH into WIN11-1 to work on the aipower app itself (not the tunnel) — including the OpenSSH-vs-plink `domain\user` login gotcha, finding the embedded MariaDB's port, and restarting Tomcat correctly — see the `aipower-lab-201-instance` skill.

Both WIN11-1 and SRV run genuinely independent `cloudflared` service processes pointed at the **same** `credentials-file` (same tunnel ID) — Cloudflare's edge load-balances across whichever connector(s) are up. This means **the two machines' `config.yml` are deliberately different**: same hostnames/ingress logic, but service addresses differ because SRV has to reach the origin over the LAN while WIN11-1 talks to it via `127.0.0.1`. When editing ingress rules, **change both files**, adjusting the IP per machine, and restart both services.

Check current live connectors from any machine that has `cloudflared` + the account's `cert.pem`:
```powershell
$cf = "C:\Program Files (x86)\cloudflared\cloudflared.exe"
& $cf tunnel info hch   # lists each connected connector with its ID + edge locations
```

## Deep Freeze wipes this laptop's cloudflared on every reboot

This laptop runs Deep Freeze (resets `C:\` on reboot — see `deepfreeze-symlink-restore` skill). `cloudflared`'s install (`C:\Program Files (x86)\cloudflared\`), its Windows service registration, and `C:\Users\HCH\.cloudflared\` (config + credentials + `cert.pem`) all live on `C:\`, so **a reboot silently kills this laptop's connector** — no error, it just stops showing up in `tunnel info hch`. This already happened once (discovered 2026-08-15: `Get-Service cloudflared` returned nothing, no `.cloudflared` folder, despite this skill previously documenting it as fully configured). If asked to check/fix the tunnel and this laptop's piece looks entirely absent, that's the likely cause, not a real outage — check WIN11-1/SRV's connectors first before assuming the whole tunnel is down.

**Don't casually reboot this laptop while triaging a cloudflared problem on it** — if `cloudflared` state here is broken/half-installed, a reboot won't fix it, it'll erase all trace of the attempt (including any registry patches, `cert.pem`, credentials file) and you start over from zero next boot too.

This laptop's connector is currently **not restored** (left broken on purpose after a failed same-day repair attempt — see "Known unkillable zombie-service incident" below). If you need it back, redo the "Fresh Install" steps below for `C:\Users\HCH\.cloudflared\`, pointing ingress at `127.0.0.1:<port>` for whatever actually runs on this laptop.

## Known gap: cbm-lite (port 12621) location is currently unknown

The original ingress config (still the right shape, see below) routes `/liff`, `/agent`, `/gateway` paths and the `gateway.james-huang.org`/`cs.james-huang.org` hostnames to `cbm-lite` on port 12621. As of 2026-08-15, **12621 is confirmed NOT listening on WIN11-1, SRV, or this laptop** (checked all three). Where it actually runs is unresolved — those ingress rules were deliberately **omitted** from both WIN11-1's and SRV's current `config.yml` rather than guessed at, so the LINE webhook / LIFF / agent console are currently unreachable through this tunnel. Find cbm-lite before restoring those rules — see `cbm-lite` skill for what it is; this skill doesn't cover finding/starting it.

## Current Ingress — WIN11-1 (primary, actual origin)

`C:\Users\administrator\.cloudflared\config.yml` on `192.168.2.201`:
```yaml
tunnel: hch
credentials-file: C:\Users\administrator\.cloudflared\fca39237-d676-470f-ad9b-4fe870d94026.json

ingress:
  - hostname: hch.james-huang.org
    service: http://127.0.0.1:22821        # aipower Tomcat (ECP)
  - service: http_status:404
```

## Current Ingress — SRV (secondary/HA connector)

`C:\Users\Administrator\.cloudflared\config.yml` on `192.168.2.200`:
```yaml
tunnel: hch
credentials-file: C:\Users\Administrator\.cloudflared\fca39237-d676-470f-ad9b-4fe870d94026.json

ingress:
  - hostname: hch.james-huang.org
    service: http://192.168.2.201:22821    # WIN11-1's LAN IP, NOT 127.0.0.1 — SRV doesn't run aipower itself
  - service: http_status:404
```

## Full ingress shape once cbm-lite is found again (reference — not currently live)

This was the last known-working shape when cbm-lite was reachable. Restore this pattern (adjusting the IP per machine as above) once cbm-lite's location is confirmed:
```yaml
  - hostname: hch.james-huang.org
    path: ^/liff                           # path 分流：LIFF 走 cbm-lite
    service: http://<cbm-lite-host>:12621
  - hostname: hch.james-huang.org
    path: ^/agent                          # cbm-lite 客服台主控台 UI
    service: http://<cbm-lite-host>:12621
  - hostname: hch.james-huang.org
    path: ^/gateway                        # LINE webhook 實際登記的 URL 就是這個 host+path
    service: http://<cbm-lite-host>:12621
  - hostname: hch.james-huang.org
    service: http://127.0.0.1:22821        # or 192.168.2.201:22821 on SRV — catch-all, keep LAST among hch.james-huang.org rules
  - hostname: gateway.james-huang.org
    service: http://<cbm-lite-host>:12621  # cbm-lite (LINE webhook 別名 host)
  - hostname: cs.james-huang.org
    service: http://<cbm-lite-host>:12621  # cbm-lite (LINE webhook 別名 host)
  - service: http_status:404
```
> **主要對外網域是 `hch.james-huang.org`**（LINE 後台實際登記的 Webhook URL 是 `https://hch.james-huang.org/gateway`）。`gateway.james-huang.org`/`cs.james-huang.org` 是備用別名 host。**path 規則要排在同 host 的 catch-all 之前**（top-to-bottom, first match wins）。

Verify path-based routing without restarting:
```powershell
$cf = "C:\Program Files (x86)\cloudflared\cloudflared.exe"
& $cf tunnel ingress validate
& $cf tunnel ingress rule https://hch.james-huang.org/liff   # should match the 12621 rule
& $cf tunnel ingress rule https://hch.james-huang.org/       # should match the 22821 catch-all
```

## Fresh Install / Adding Another Machine as a Connector

You do **not** need to run `cloudflared tunnel login` on the target machine itself. The login/auth step only needs *a* browser session on *any* machine with access to the Cloudflare account — the resulting `cert.pem` and a freshly-minted credentials JSON can then be copied to wherever the new connector needs to run (this is how WIN11-1 and SRV were both set up in the same session without ever getting a GUI/browser onto either VM):

```powershell
# On any machine with a browser (e.g. this laptop):
winget install --id Cloudflare.cloudflared -e --source winget --accept-package-agreements --accept-source-agreements
$cf = "C:\Program Files (x86)\cloudflared\cloudflared.exe"
& $cf tunnel login                      # prints a URL — open it in Chrome, authorize the james-huang.org zone
& $cf tunnel list                       # confirm "hch" (fca39237-d676-470f-ad9b-4fe870d94026) is listed
& $cf tunnel token --cred-file "<some local temp path>\fca39237-d676-470f-ad9b-4fe870d94026.json" hch
```
`tunnel token --cred-file` re-derives a **valid, fresh** credentials file for the *existing* tunnel — it does not create a new tunnel and does not require the original credentials file to still exist. Safe to re-run any time you need another copy (e.g. one per new connector machine), from any machine that has done `tunnel login` for that account.

Then, on the **target** machine (via SSH if remote):
```powershell
winget install --id Cloudflare.cloudflared -e --source winget --accept-package-agreements --accept-source-agreements
# (if winget's App Execution Alias isn't visible in a non-interactive SSH session — seen on WIN11-1 —
#  download the binary directly instead: https://github.com/cloudflare/cloudflared/releases/latest ,
#  grab cloudflared-windows-amd64.exe, place it at "C:\Program Files (x86)\cloudflared\cloudflared.exe")
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.cloudflared"
# copy the credentials JSON here (scp/pscp/SSH-piped write — it's a secret, byte-for-byte, don't print it)
# write config.yml here (ingress pointing at wherever the real origin service actually is — 127.0.0.1
#  only if the service runs on THIS SAME machine, otherwise its real LAN IP)
& $cf tunnel ingress validate
```

Then install the service — **read the gotcha below before you do**, the exact command sequence matters.

### Gotcha: `cloudflared service install` on Windows does NOT wire up your config

`cloudflared.exe service install` registers the Windows service with **`LocalSystem`** as the logon account and **no arguments** in `ImagePath`. There is no `--config` flag on `service install` itself. At runtime the service falls back to the default config lookup path relative to the `LocalSystem` profile, **not** your actual `.cloudflared` folder. Result: the service shows `Running` in `Get-Service`, but `cloudflared tunnel info hch` reports no connection for it, `http://127.0.0.1:20241/ready` refuses to connect, and the public hostname returns **530**. No obvious error anywhere (Application event log only shows generic "service starting" lines) — silent, not a crash.

**Patch `ImagePath` BEFORE the first `Start-Service`, not after — ordering matters, not just doing it eventually:**
```powershell
# 1. Install (elevated) — do NOT start it yet
& $cf service install

# 2. Patch ImagePath immediately, before any Start-Service call (elevated)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\cloudflared" -Name ImagePath `
  -Value '"C:\Program Files (x86)\cloudflared\cloudflared.exe" --config "<full path to config.yml>" tunnel run'

# 3. Only now start it
Start-Service cloudflared
```
Adding `tunnel run` + `--config` to `ImagePath` does not break SCM registration — cloudflared still detects it's running under the Service Control Manager, still responds correctly to stop/start.

**Why the ordering matters — a confirmed unkillable-zombie failure mode:** `cloudflared.exe service install` can auto-start the service (with the stale no-`--config` `ImagePath`) *during* the install itself, before your patch script's `Set-ItemProperty` line even runs. If you patch-then-restart in the wrong order (install → start → patch → stop/restart), the **first** (misconfigured) process instance can end up stuck in a Windows service state that even elevated-Administrator `taskkill /F` cannot terminate (`Access is denied`, service permanently wedged at `STATE: STOP_PENDING`, `CHECKPOINT: 0x0`, never progresses). This happened for real on this laptop on 2026-08-15 and was never fully recovered in that session — deleting/reinstalling the service didn't help because the zombie process kept the old registration alive. **The only thing that worked elsewhere in this environment for an equivalent "Administrator can't kill a stuck LocalSystem process" problem was a one-off SYSTEM-context Scheduled Task** (same trick as the foreground-`sshd` workaround in `esxi-python-control` skill):
```powershell
$script = @'
taskkill /F /IM cloudflared.exe /T
sc.exe delete cloudflared
# ... reinstall + patch ImagePath BEFORE Start-Service, then Start-Service ...
'@
Set-Content -Path "$env:TEMP\fix_cloudflared.ps1" -Value $script -Encoding UTF8
$action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument "-NoProfile -ExecutionPolicy Bypass -File `"$env:TEMP\fix_cloudflared.ps1`""
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -RunLevel Highest
Register-ScheduledTask -TaskName "FixCloudflaredOnce" -Action $action -Principal $principal
Start-ScheduledTask -TaskName "FixCloudflaredOnce"
```
Registering/starting the scheduled task itself still needs an elevated (UAC) context to run `Register-ScheduledTask`/`Start-ScheduledTask` — wrap that part in `Start-Process powershell -Verb RunAs -Wait`. **Best fix is to avoid this entirely by never starting the service until after `ImagePath` is patched** (see step-by-step above) — this failure mode was *not* hit on WIN11-1 or SRV once the install→patch→start ordering was followed strictly, only on the laptop where install→start→patch→restart was tried first.

**Verify it actually loaded the config (don't trust `Get-Service` status alone):**
```powershell
& $cf tunnel info hch                                          # should list this connector, not omit it
Invoke-WebRequest http://127.0.0.1:20241/ready -UseBasicParsing # should be 200
Get-CimInstance Win32_Process -Filter "Name='cloudflared.exe'" | Select-Object ProcessId,CommandLine  # confirm --config is actually in the running process's args, not just the registry
```

**If `net stop cloudflared` / `Stop-Service` hangs in `STOP_PENDING`** (seen when the process already has live tunnel connections — not always, sometimes a stop/restart completes cleanly): don't wait it out. Force-kill and restart:
```powershell
taskkill /F /IM cloudflared.exe /T
Start-Sleep 2
Start-Service cloudflared    # or: net start cloudflared
```
If even the force-kill gets `Access is denied`, that's the unkillable-zombie case above — needs the SYSTEM scheduled task.

## Service Management

```powershell
# Check status
Get-Service cloudflared | Select-Object Name, Status

# Restart (requires UAC elevation)
Start-Process powershell -ArgumentList "-NoProfile -Command 'net stop cloudflared; Start-Sleep 2; net start cloudflared'" -Verb RunAs -Wait
Start-Sleep -Seconds 5
(Get-Service cloudflared).Status
```

**Important:** After editing `config.yml`, the service must be restarted to pick up new ingress rules — it loads config once at startup. If this is a multi-machine ingress change, restart **both** WIN11-1's and SRV's services, not just one.

## Add a New Hostname Route

Two steps:

### Step 1 — Add DNS CNAME in Cloudflare dashboard
In Cloudflare DNS: add CNAME `<subdomain>` → `fca39237-d676-470f-ad9b-4fe870d94026.cfargotunnel.com` (Proxied).

Or via CLI:
```powershell
$cf = "C:\Program Files (x86)\cloudflared\cloudflared.exe"
& $cf tunnel route dns hch <subdomain>.james-huang.org
```

### Step 2 — Add ingress rule in config.yml on every machine running a connector for this tunnel
```yaml
  - hostname: <subdomain>.james-huang.org
    service: http://<ip-reachable-from-that-machine>:<PORT>
```
Then restart the service **on each connector machine** you edited.

## Verify

```powershell
# Local service first
Invoke-RestMethod http://127.0.0.1:<PORT>/health

# Through tunnel
Invoke-RestMethod https://<subdomain>.james-huang.org/health
```

If testing from a machine whose own internet egress is flaky/broken (this laptop had that problem separately from the tunnel itself on 2026-08-15 — DNS resolved fine but the request timed out), don't conclude the tunnel is down from that alone. Cross-check from a different machine (e.g. one of the connector VMs itself) before troubleshooting the tunnel/origin.

## Troubleshooting: 404 from external URL

**Cause:** cloudflared is running with old config (before new hostname was added to ingress).
**Fix:** Restart cloudflared service (needs admin/UAC) — on every connector machine.

The last catch-all rule `service: http_status:404` returns 404 for any hostname not listed in ingress — this is often mistaken for a "service is down" error.

**Common variant:** a specific *subpath* under an already-working hostname returns 404 while the rest of the hostname works fine. Cause: that hostname only has a catch-all rule pointing at service A, but the actual endpoint being hit (e.g. LINE webhook at `/gateway`) belongs to service B — needs its own explicit `path:` rule added **before** the catch-all. Symptom looks exactly like "webhook can't connect" / "404 Not Found" from the LINE Developers Console's Verify button.

## Gotcha: "Verify" button (LINE, or similar webhook consoles) only proves connectivity + signature, not business logic

LINE's Webhook "Verify" button sends a payload with an **empty `events` array** (`{"events":[]}`) — it only proves: URL is reachable, TLS/tunnel routing works, and (if the receiving server checks it) the signature validates. It does **not** exercise the actual message-handling code path (parsing a real `message`/`text` event, DB writes, etc.).

Symptom this causes: "Verify → Success" and a curl test with `{"events":[]}` both return 200, yet real user messages never get processed/stored — because the bug is in the code that only runs for non-empty, real events. To actually test the business logic, send a **realistic simulated payload** with a real `message` event (correct HMAC-SHA256 signature over the exact request body using the channel secret) and check the receiving side's data store directly, not just the HTTP status code.

## Delegating multi-machine setup work

This kind of task (SSH into 2+ remote Windows VMs, install/configure a service, verify) is a good fit for handing off to a `pi` agent one-shot call (`pi --provider anthropic --model claude-sonnet-5 -p "<prompt>" --no-session`, see `pi-omni-model-test` skill) rather than driving every SSH command directly — it worked cleanly for both WIN11-1 and SRV in this session. Two practical notes from doing it:
- **Write the prompt to a file first, then pass it via `$(cat file)`**, rather than inlining a long prompt directly in a Bash heredoc/command string. A prompt containing ordinary English contractions (`isn't`, `doesn't`, `wasn't`) broke the Bash tool's command parsing here (`unexpected EOF while looking for matching` an unmatched single quote) — writing the same text to a file with the `Write` tool and reading it back sidesteps the issue entirely.
- Give the delegated agent the **exact same gotcha writeups from this skill** inline in its prompt (the `ImagePath`-before-`Start-Service` ordering, the zombie-service risk) — it hit and correctly self-recovered from the "install auto-started with stale ImagePath" race on WIN11-1 because it had been warned what to watch for and how to fix it, without needing to escalate back.

---

## Conformance Addendum

## When to Use
Manage the multi-machine Cloudflare Tunnel "hch" (cloudflared) — connector status across WIN11-1/SRV/this laptop, config, restart service, add hostname routes, and the Windows service-install gotchas

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
