---
name: voicemeeter-remote-api
description: Use when controlling VB-Audio Voicemeeter (Standard/Banana/Potato) programmatically from Python/ctypes or any language — muting/unmuting strips or buses, setting gain/volume, reading levels, or automating via the official VoicemeeterRemote DLL. Also use when Set calls report success (return 0) but Get immediately reads back the old value, or gain lands on a strange transitional number.
---

# Voicemeeter Remote API (ctypes)

## Overview

Voicemeeter installs `VoicemeeterRemote64.dll` (and 32-bit `VoicemeeterRemote.dll`) that any
language can call via FFI/ctypes to read or write every mixer parameter. No extra package
needed — raw ctypes is enough.

Typical DLL path: `C:\Program Files (x86)\VB\Voicemeeter\VoicemeeterRemote64.dll`

## Two gotchas that will burn you

1. **Must declare `argtypes`/`restype` on the ctypes functions.** Without them, ctypes
   guesses the wrong calling convention for `c_float` args on x64 and `SetParameterFloat`
   silently no-ops (returns 0 = "success" but the value never changes).

2. **The local parameter mirror is stale until you poll `VBVMR_IsParametersDirty()`.**
   After `Login()` and after any `Set`, call `IsParametersDirty()` before trusting a `Get` —
   otherwise you'll read the pre-change value even though the set "succeeded". Gain changes
   also **fade internally** (~0.5–1.5s ramp) — read too soon after setting Gain and you'll see
   a transitional number (e.g. asked for +12, read back -7.8) instead of the final value. Wait
   ~1s (or poll `IsParametersDirty` in a loop) before reading gain back.

## Quick Reference: Strip/Bus indices (Banana)

| Object | Count | Index | Notes |
|---|---|---|---|
| `Strip[i]` | 5 | 0-4 | 0-2 = hardware inputs, 3-4 = virtual inputs |
| `Bus[i]` | 5 | 0-4 | 0-2 = A1/A2/A3 (physical out), 3-4 = B1/B2 (virtual out) |

Common parameters: `Mute` (0/1), `Gain` (-60 to +12 dB), `Mono` (0/1), `Solo` (0/1),
`A1..A5`/`B1..B3` (routing, 0/1), `Label` (string), `Comp`, `Gate`, `Karaoke`, `EQGain1-3`.

## Working example

```python
import ctypes, time

DLL = r"C:\Program Files (x86)\VB\Voicemeeter\VoicemeeterRemote64.dll"
vm = ctypes.CDLL(DLL)

# REQUIRED: declare signatures or float sets will silently no-op
vm.VBVMR_Login.restype = ctypes.c_long
vm.VBVMR_Logout.restype = ctypes.c_long
vm.VBVMR_IsParametersDirty.restype = ctypes.c_long
vm.VBVMR_SetParameterFloat.argtypes = [ctypes.c_char_p, ctypes.c_float]
vm.VBVMR_SetParameterFloat.restype = ctypes.c_long
vm.VBVMR_GetParameterFloat.argtypes = [ctypes.c_char_p, ctypes.POINTER(ctypes.c_float)]
vm.VBVMR_GetParameterFloat.restype = ctypes.c_long

vm.VBVMR_Login()
vm.VBVMR_IsParametersDirty()  # prime the mirror

# mute all 5 strips + 5 buses
for i in range(5):
    vm.VBVMR_SetParameterFloat(f"Strip[{i}].Mute".encode(), ctypes.c_float(1.0))
    vm.VBVMR_SetParameterFloat(f"Bus[{i}].Mute".encode(), ctypes.c_float(1.0))

time.sleep(0.3)              # let mute settle
vm.VBVMR_IsParametersDirty() # refresh mirror before reading back

v = ctypes.c_float()
vm.VBVMR_GetParameterFloat(b"Strip[0].Mute", ctypes.byref(v))
print(v.value)  # 1.0

vm.VBVMR_Logout()
```

For gain, use `time.sleep(1.0)` (not 0.3s) before reading back — the internal fade takes
longer than mute's on/off flip.

## Detecting install / version

```python
vtype = ctypes.c_long()
vm.VBVMR_GetVoicemeeterType(ctypes.byref(vtype))
# 1=Standard, 2=Banana, 3=Potato

ver = ctypes.c_long()
vm.VBVMR_GetVoicemeeterVersion(ctypes.byref(ver))
v = ver.value
print(f"{(v>>24)&0xFF}.{(v>>16)&0xFF}.{(v>>8)&0xFF}.{v&0xFF}")
```

`Login()` returns `0` = ok, `1` = ok but Voicemeeter app isn't running (mixer still works via
driver-only mode), negative = error.

## Common Mistakes

| Symptom | Fix |
|---|---|
| `SetParameterFloat` returns 0 but value never changes | Declare `argtypes`/`restype` before calling |
| Gain reads a weird transitional value right after set | Wait ~1s (fade animation), or poll `IsParametersDirty()` until it returns 0 |
| `Get` returns stale value after a successful `Set` | Call `VBVMR_IsParametersDirty()` once before the `Get` |
| Don't know max/min gain | -60 to +12 dB, hard-clamped by the app |

---

## Conformance Addendum

## When to Use
Use when controlling VB-Audio Voicemeeter (Standard/Banana/Potato) programmatically from Python/ctypes or any language — muting/unmuting strips or buses, setting gain/volume, reading levels, or automating via the official VoicemeeterRemote DLL. Also use when Set calls report success (return 0) but Get immediately reads back the old value, or gain lands on a strange transitional number.

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
