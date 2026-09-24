# src/xrSound/SoundRender_CoreA.h

> Declares the device-backed sound manager, and the error-checking wrappers that surround every
> mixer call in a debug build.

**Needs** — [`SoundRender_Core.h`](SoundRender_Core.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`Sound.cpp`](Sound.cpp.md) · [`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md) · [`SoundRender_TargetA.h`](SoundRender_TargetA.h.md)
**Tier floor** — T1: it names device, context and format handle types.

## Purpose

Declares the backend implemented in [`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md). Together
with [`SoundRender_TargetA.h`](SoundRender_TargetA.h.md) this is the entire device seam's concrete
side.

## Exported units

- **`DeviceSoundCore`** — holds the device handle, its context, and the enumerated device list.
- The five operations it fills in: build the device list, initialize, clear, restart, set master
  volume; plus the listener update it extends.
- **Checked-call wrappers** — in a debug build each mixer call is bracketed by clearing and then
  reading the mixer's error state, and a non-clear state fails loudly with the device's own message;
  in a shipping build the call is made bare. A rebuild needs *some* equivalent, because a mixer
  that fails silently leaves no trace at all — but the checking must be compiled out of the
  shipping path, since it doubles the call count on a per-voice-per-frame path.
- **Float-PCM format fallback** — where the mixer's headers predate the float extension, the two
  format identifiers are defined locally to their known values. They are wire constants of the
  extension, not choices.
