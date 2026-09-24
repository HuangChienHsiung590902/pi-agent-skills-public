---
name: rainmeter-remote-gpu-monitor
description: Build, install, repair, or update a Windows Rainmeter desktop dashboard that shows NVIDIA GPU stats (utilization, memory, temp, power, fan, processes) from a remote Linux host reached via passwordless SSH, styled like nvitop. Use when the user mentions Rainmeter plus GPU/nvidia-smi/nvitop monitoring, wants a remote-host GPU dashboard, or Rainmeter RunCommand measures showing blank/empty values.
compatibility: Windows, Rainmeter, OpenSSH client (ssh.exe), remote host with nvidia-smi in PATH and passwordless SSH key auth already configured.
---

# Rainmeter Remote GPU Monitor (nvitop-style)

Rainmeter skin that polls a remote NVIDIA GPU host over SSH and renders
nvitop-style bars (utilization, memory, temperature, power, fan, process list)
on the Windows desktop. Rainmeter cannot embed a real TUI (nvitop itself is a
curses full-screen app and cannot be shown inside a Rainmeter meter), so this
skin re-implements the same numbers as flat progress bars using `nvidia-smi
--query-gpu=...,--format=csv`.

Installed/default skin path:

```text
%USERPROFILE%\Documents\Rainmeter\Skins\GPUMonitor\GPUMonitor.ini
```

Bundled template path, relative to this skill:

```text
templates\GPUMonitor\
```

Known-good example target host (edit if different): `hch@10.145.119.19`,
2x NVIDIA GeForce RTX 4060 Ti, `nvidia-smi` in PATH, SSH key auth already
works (no ssh-copy-id needed there).

## When to use

Use this skill when the user asks to:

- show remote/local GPU usage, memory, temperature, power, fan on the desktop via Rainmeter
- replicate `nvitop` visuals without an actual terminal
- fix a Rainmeter GPU skin where bars/numbers show blank, or the meter shows
  literal file paths instead of numbers
- change refresh interval of a GPU dashboard
- add a second/third GPU or a process list

## Architecture

```
Rainmeter (Windows) --UpdateRate--> fetch_gpu.bat --ssh--> remote host
                                          |
                                          v
                      g0_util.txt, g0_memused.txt, g0_temp.txt, ... (per-GPU scalar files)
                      proc_data.txt (process list)
                                          |
                                          v
                      Rainmeter RunCommand measures (`cmd /c type <file>`)
                                          |
                                          v
                      Bar / String meters
```

Rainmeter's `RunCommand` plugin captures **stdout of the command it runs** as
the measured value. So instead of parsing one multi-column CSV with
`WebParser` (which may be missing on some Rainmeter installs — see pitfalls),
`fetch_gpu.bat` pre-splits `nvidia-smi` CSV output into one plain-text file
per metric per GPU, and each Rainmeter measure just does
`cmd /c type <that file>` to read the number back. This avoids depending on
the `WebParser` plugin entirely.

## Prerequisites — do this FIRST

1. Windows OpenSSH client available: `ssh -V` in PowerShell.
2. Passwordless SSH key auth to the remote host already works:

   ```powershell
   ssh-keygen -t ed25519          # if no key yet
   type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@<host> "cat >> ~/.ssh/authorized_keys"
   ssh <user>@<host> "nvidia-smi -L"   # must NOT prompt for a password
   ```

3. Verify remote `nvidia-smi` output shape before writing any INI:

   ```bash
   ssh <user>@<host> "nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw,power.limit,fan.speed --format=csv,noheader,nounits"
   ```

   Count how many GPUs are returned (one line per GPU) — the skin needs one
   block of measures/meters per GPU index.

## Install / reinstall

```bash
mkdir -p "$USERPROFILE/Documents/Rainmeter/Skins/GPUMonitor"
cp templates/GPUMonitor/GPUMonitor.ini templates/GPUMonitor/fetch_gpu.bat \
   "$USERPROFILE/Documents/Rainmeter/Skins/GPUMonitor/"
```

Edit `fetch_gpu.bat`'s `set REMOTE=user@host` line if the target host differs
from the bundled default.

Then in Rainmeter: right-click tray icon → **Manage** → find `GPUMonitor` →
double-click `GPUMonitor.ini`. Or if already loaded, right-click the skin on
the desktop → **Refresh Skin**.

Do **not** try to load/refresh a skin via `Rainmeter.exe` command-line
bang commands from a shell — see Pitfalls below, it reliably corrupts
`Rainmeter.ini` write attempts under `C:\Program Files\Rainmeter\` if that
install lacks write permission there. Prefer the GUI (Manage window,
right-click → Refresh Skin) or ask the user to do it.

## Change refresh interval

Two places must match in `GPUMonitor.ini`:

```ini
[Rainmeter]
Update=2000        ; global skin update tick, ms

[MeasureRunSSH]
UpdateRate=1        ; multiples of the global Update above
```

`UpdateRate=1` on `MeasureRunSSH` means it runs every tick (every `Update` ms).
Each fetch does 1-2 SSH round trips; measured latency to the known-good host
is ~0.9s for both `nvidia-smi` calls combined. Do not set `Update` below
~2000ms unless the remote SSH round-trip is confirmed much faster — going too
fast can pile up overlapping `ssh` processes.

All other per-metric measures (`MeasureUtil0`, `MeasureTemp0`, ...) read
already-fetched local text files with `cmd /c type`, so they are cheap and can
have `UpdateRate=1` safely; they just reflect whatever `fetch_gpu.bat` last
wrote.

## Add a second/third GPU

1. Confirm `nvidia-smi --query-gpu=...` on the remote host returns one line
   per GPU index (0, 1, 2, ...).
2. `fetch_gpu.bat` already loops over every returned line with a
   `for /f "tokens=1-9 delims=," %%a in (...)` batch loop and writes
   `g<index>_*.txt` files automatically for however many GPUs are present —
   no changes needed there for more GPUs.
3. In `GPUMonitor.ini`, duplicate the whole "GPU 1" measures+meters block,
   renaming `0`/`1` suffixes to the new GPU index (e.g. `2`), and adjust each
   meter's `Y=` coordinate to stack below the previous GPU block (~88px per
   GPU block is enough).

## Test the fetch script manually (fastest way to isolate bugs)

```bash
cd "$USERPROFILE/Documents/Rainmeter/Skins/GPUMonitor"
cmd //c fetch_gpu.bat
cat g0_util.txt g0_temp.txt g0_memused.txt g0_memtotal.txt g0_power.txt g0_powerlimit.txt g0_fan.txt g0_name.txt
cat proc_data.txt
```

If these files contain correct plain numbers, the SSH/batch layer is fine and
any remaining "blank/wrong" symptom in Rainmeter is an INI/plugin-syntax bug,
not a data problem — go straight to Pitfalls below instead of re-touching SSH.

## Pitfalls encountered (read before debugging blank values)

### 1. `RunCommand` plugin syntax: `Program=` and `Parameter=` MUST be separate keys

Wrong (silently fails / all measures render blank, though the process runs
and produces correct output when tested in a shell):

```ini
Plugin=RunCommand
Parameter=cmd /c type "#@#g0_util.txt"
```

Correct:

```ini
Plugin=RunCommand
Program=cmd
Parameter=/c type "#@#g0_util.txt"
```

This bit twice in the same debugging session — always grep the INI for
`Parameter=cmd` and split it into `Program=cmd` + `Parameter=/c ...` if found.

### 2. `FileView` plugin is NOT a file-content reader

It's easy to assume `Plugin=FileView` with `Path=...` reads a text file's
content into the measure value. It does not — it's meant for folder/file
*listing* (browsing), and using it this way makes the meter display the
literal file path string instead of the file's numeric content. Use
`RunCommand` with `cmd /c type "<file>"` instead (see architecture above).

### 3. `WebParser` plugin may not be installed on all Rainmeter setups

Check first:

```bash
ls "/c/Program Files/Rainmeter/Plugins/" | grep -i webparser
```

If missing, don't rely on `WebParser` + `RegExp` to parse multi-field CSV in
one shot — use the pre-split-into-scalar-files approach in this skill's
`fetch_gpu.bat` instead, which only needs `RunCommand`.

### 4. Never drive Rainmeter.exe with ad-hoc bang commands from a shell for skin (re)load

Calling something like:

```bash
"/c/Program Files/Rainmeter/Rainmeter.exe" '[!ActivateConfig "GPUMonitor" "GPUMonitor.ini"]'
```

from a bash/mintty shell is fragile: quoting/escaping gets mangled by the
shell, Rainmeter can misparse the bang command as a bogus folder name, and if
the install is under `C:\Program Files\Rainmeter\` without write permission,
Rainmeter pops a blocking `Rainmeter.ini 檔案無法寫入` (cannot write
Rainmeter.ini) error dialog that must be manually dismissed by the user
before anything else works. Prefer: ask the user to use the Rainmeter tray
icon → Manage window, or right-click the loaded skin → Refresh Skin.

### 5. Batch file comments with non-ASCII (e.g. Chinese) text can break `cmd.exe`

A `REM 中文註解` line in `fetch_gpu.bat` produced
`'中文...' is not recognized as an internal or external command` errors when
run under some codepages, even though it's just a comment. It doesn't break
the actual data fetch (stdout redirection to the CSV file still works), but
it's noisy/confusing. Keep batch file comments ASCII-only.

### 6. Windows batch `nounits` CSV values have leading spaces

`nvidia-smi ... --format=csv,noheader,nounits` output is comma-space
separated (`0, NVIDIA GeForce RTX 4060 Ti, 29, 4546, ...`), so each `%%b`,
`%%c`, ... token captured by a batch `for /f ... delims=,` loop retains a
leading space (e.g. `" NVIDIA GeForce RTX 4060 Ti"`, `" 29"`). Rainmeter's
Bar/String meters tolerate this fine when the value is purely numeric
(`Calc`/`Bar` measures trim/parse), but don't be surprised seeing a leading
space if you `cat` a `g0_*.txt` file directly.

## Files in this skill's template

- `GPUMonitor.ini` — the Rainmeter skin: RunCommand measures (SSH fetch +
  per-metric file reads), Calc measures for %, Bar/String/Shape meters.
- `fetch_gpu.bat` — SSH's into the remote host twice (once for
  `nvidia-smi --query-gpu`, once for `nvidia-smi --query-compute-apps`),
  splits the GPU CSV per-index into scalar `.txt` files, ASCII-only comments.

## Troubleshooting checklist

1. `cmd //c fetch_gpu.bat` manually in the skin folder — do the `g*_*.txt`
   files contain correct plain numbers? If not, it's an SSH/remote problem,
   not Rainmeter (check passwordless auth, remote `nvidia-smi` PATH).
2. If files are correct but Rainmeter meters are blank — check every
   `Plugin=RunCommand` measure has `Program=` + `Parameter=` as separate
   keys (Pitfall 1).
3. If meters show a literal path instead of a number — some measure was
   accidentally wired to `Plugin=FileView` (Pitfall 2); switch it to
   `RunCommand` + `cmd /c type`.
4. If Rainmeter pops a `Rainmeter.ini 檔案無法寫入` dialog — that's from an
   ad-hoc command-line bang-command attempt (Pitfall 4); tell the user to
   click 確定 (OK) to dismiss, then reload the skin via the GUI instead.

---

## Conformance Addendum

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
