---
name: tiny11-build
description: This skill should be used when the user asks to build a trimmed/debloated Windows 11 ISO, mentions "tiny11", "tiny11builder", "精簡版 win11", "build a stripped down Windows 11 image", or wants to customize/rebuild the ISO at D:\tiny11builder-main with a source Windows 11 ISO.
---

# Build a trimmed Windows 11 ISO with tiny11builder

Drives `D:\tiny11builder-main` (the ntdevlabs/tiny11builder PowerShell scripts) to
turn a stock Windows 11 ISO into a debloated one, using a copy of the scripts that
already has custom patches applied (see "Custom patches already applied" below).

## Key facts

- Every DISM operation (`Mount-WindowsImage`, `Get-WindowsImage`, `dism.exe`
  itself) **requires admin elevation**, even just to read image info. The
  Claude Code shell in this environment is **not** elevated and cannot click
  through a UAC prompt, so the actual build **must be run by the user** in
  their own "Run as administrator" PowerShell window — never attempt to
  self-elevate and run it via the Bash/PowerShell tool.
- `tiny11maker.ps1` also needs interactive input mid-run (it lists the
  Windows editions found in `install.wim` and does `Read-Host` for the index
  to build) — another reason it can't run unattended from here.
- Two script variants exist:
  - `tiny11maker.ps1` (recommended default) — keeps WinSxS/Windows
    Update/WinRE, image stays serviceable (can add languages/updates/features
    later).
  - `tiny11Coremaker.ps1` — also strips WinSxS, Windows Update, WinRE; image
    is no longer serviceable. Only offer this if the user explicitly wants a
    throwaway/dev/VM image.
- Mounting the source ISO with `Mount-DiskImage` does **not** require
  elevation and can be done directly from this shell.

## Custom patches already applied in D:\tiny11builder-main

These are one-off edits layered on top of the upstream scripts. If the repo
ever gets re-cloned/reset, they need to be reapplied (ask the user before
reapplying — don't silently assume upstream still matches).

1. **`tiny11maker.ps1`** — Windows Defender fully removed (not just
   disabled): after the OneDrive removal block, it now enumerates packages
   via `dism /Get-Packages` and runs `/Remove-Package` on
   `Windows-Defender-Client-Package~31bf3856ad364e35~*`. Later, in the
   registry-loaded section (near the "Prevent installation of New Outlook"
   line), it also sets `WinDefend`, `WdNisSvc`, `WdNisDrv`, `WdFilter`,
   `Sense` service `Start=4`, sets `DisableAntiSpyware=1`, and hides the
   "virus & threat protection" settings page.
2. **`autounattend.xml`** — added a `pass="specialize"` block
   (`Microsoft-Windows-Deployment` / `RunSynchronous`) that on first boot:
   waits up to ~60s for network, then `Add-WindowsCapability -Online -Name
   OpenSSH.Server~~~~0.0.1.0`, sets `sshd` to `Automatic` and starts it, and
   adds an inbound firewall rule for TCP 22. All three steps swallow errors
   (`try/catch; exit 0`) so a network-less first boot degrades gracefully
   instead of failing setup — it just means OpenSSH won't be installed and
   the user has to run the three commands manually later (see below).
   This file is used both as the ISO-root autounattend and copied into the
   image's `Windows\System32\Sysprep\` by `tiny11maker.ps1` itself, so no
   extra wiring is needed for either script variant.
   - **Caveat to tell the user**: the specialize pass runs *before* OOBE's
     network-setup page, which is bypassed anyway (`BypassNRO`), so this only
     succeeds if the machine has working wired/DHCP network at first boot.
     Manual fallback if it didn't install:
     ```powershell
     Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
     Set-Service sshd -StartupType Automatic; Start-Service sshd
     netsh advfirewall firewall add rule name="OpenSSH Server (TCP-In)" dir=in action=allow protocol=TCP localport=22
     ```

## Procedure

1. Confirm the source ISO path and which script variant to use (default to
   `tiny11maker.ps1` unless the user wants the non-serviceable Core variant —
   ask if unclear, same tradeoff as documented in the repo's README).
2. Mount the source ISO (no elevation needed):
   ```powershell
   Mount-DiskImage -ImagePath "<path to source .iso>" -PassThru | Get-Volume
   ```
   Note the resulting drive letter.
3. Check free space on the intended scratch drive (build needs room for a
   full copy of the ISO contents plus a mounted WIM; tens of GB is safe):
   ```powershell
   Get-Volume | Where-Object {$_.DriveLetter} | Select-Object DriveLetter, @{n='FreeGB';e={[math]::Round($_.SizeRemaining/1GB,1)}}
   ```
4. Hand the user the exact commands to run themselves in an elevated
   PowerShell window (do not attempt to run these directly):
   ```powershell
   Set-ExecutionPolicy Bypass -Scope Process
   cd D:\tiny11builder-main
   .\tiny11maker.ps1 -ISO <mounted drive letter> -SCRATCH <scratch drive letter>
   ```
5. Tell them: it will list editions found in the image and prompt for an
   index number — pick the edition they want. The full run (mount, edit,
   cleanup, export, ISO creation) can take 20–40 minutes. Output lands at
   `D:\tiny11builder-main\tiny11.iso`; the script auto-ejects the mounted ISO
   and cleans up its scratch folders when done.
6. If they report errors or paste the edition list, use that to tell them
   what to type/fix next — don't guess at output you haven't seen.

## Verifying nothing has run yet / checking progress

Since the build happens in a window this session can't see into, check
progress via the filesystem instead of asking the user to describe it:
```bash
ls -la /d/tiny11builder-main/*.log   # Start-Transcript log, one per run
ls -la /d/tiny11builder-main/tiny11.iso
ls -la /d/tiny11/ /d/scratchdir/     # present while a build is in progress (drive letter may vary with -SCRATCH)
```
No log file and no `tiny11.iso` means the script hasn't been executed yet.

---

## Conformance Addendum

## When to Use
This skill should be used when the user asks to build a trimmed/debloated Windows 11 ISO, mentions "tiny11", "tiny11builder", "精簡版 win11", "build a stripped down Windows 11 image", or wants to customize/rebuild the ISO at D:\tiny11builder-main with a source Windows 11 ISO.

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

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
