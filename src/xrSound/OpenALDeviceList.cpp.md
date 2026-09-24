# src/xrSound/OpenALDeviceList.cpp

> Enumerate the machine's audio output devices, record what each can do, and choose one.

**Needs** — [`OpenALDeviceList.h`](OpenALDeviceList.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`OpenALDeviceList.h`](OpenALDeviceList.h.md); callers name that, not this file.
**Tier floor** — T1: it walks a double-null-terminated string list the device API returns.

## Purpose

Before a device is opened, the engine needs to know which devices exist, what each supports, and
which one to use. That is this file. It runs once, before anything else in the chapter, because the
answer is a setting the player can change.

## `enumerate`

**Contract** — Fills the device list and the settings-screen token list, and records the system
default device's name. Blocks: it opens, probes and closes *every* device on the machine. Logs the
result outside shipping builds. Leaves the list empty rather than failing when the platform offers
no enumeration at all.

```text
FUNCTION enumerate()
  IF the full-enumeration extension exists THEN
    names ← the full device list ; default ← the full-list default
  ELSE IF the basic enumeration extension exists THEN
    names ← the basic device list ; default ← the basic default
    apply the hardware-device substitution below
  ELSE
    log that enumeration is unavailable; the list stays empty

  probe_each(names)
  publish the names as settings tokens, plus a terminator
```

**Notes** — The two enumeration extensions list different things: the older one lists *drivers*, the
newer one lists actual output endpoints. Preferring the newer means the player picks a headset
rather than a driver, which is what they expect.

One platform-specific substitution survives: where the system default names a
hardware-mixing driver, it is rewritten to the software-mixing one of the same family. The reason
recorded in the source is that the hardware path on the codecs of the era burned up to 30% of a CPU
and caused frame drops. The comment notes this was already an old problem in 2000 and was still
present fifteen years later. The cost is that hardware 3D mixing is unreachable on those machines;
the engine takes that trade. A rebuild on a modern mixer will not need this and should drop it.

## `probe_each`

**Contract** — For each name in the list, open the device, make a context, ask what it supports,
record it, and close everything. Skips a device that will not open or will not give a context —
which is the point: a device that cannot be opened at probe time cannot be opened later either, so
it never reaches the settings screen.

```text
FOR EACH name IN names                   # single-null separated, double-null terminated
  device ← open(name)                    ; IF failed THEN CONTINUE
  context ← create_context(device)       ; IF failed THEN close; CONTINUE
  make context current
  # Ask the device for its *own* name, not the one we opened it by:
  # the two differ, and the device's own is what the player should see.
  actual ← device.name (full form where available)
  IF actual is non-empty THEN
    record (actual, major version, minor version)
    record the highest reverb extension generation it claims: 5, 4, 3, 2, or none
    record whether it offers the effects extension
  destroy the context; close the device
```

**Notes** — Reverb generation is recorded as a small integer rather than a flag, highest-first, so
that later code can ask "is there any reverb" and, if it wanted to, "which generation". Only the
first question is asked today.

The whole probe requires making a context current per device, which temporarily displaces any
context the process already has. That is safe only because this runs before the real device is
opened — a rebuild must preserve that ordering or restore the previous context.

## `select_best_device`

**Contract** — Chooses the device index, unless the player already chose one. Leaves an explicit
choice alone. Falls back to the first device when nothing matches, and asserts only that the list is
not empty.

```text
FUNCTION select_best_device()
  IF the player already chose a device THEN RETURN
  best ← none
  FOR EACH device whose name equals the system default
    keep the one with the highest (major, minor) version
  IF best IS none THEN best ← the first device
  chosen ← best
```

**Notes** — "Best" means *the system default, at its newest specification version* — not the
device with the most features. The engine defers to the operating system's choice of output and only
breaks ties among identically named entries. That is the right policy for a game: the player has
already told the OS which output they want.

## `device_name` / `device_version`

**Contract** — Read one device's recorded name or specification version by index. Pure.
