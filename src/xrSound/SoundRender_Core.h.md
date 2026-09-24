# src/xrSound/SoundRender_Core.h

> Declares the target-agnostic sound manager and the four operations a device backend must fill in.

**Needs** — [`SoundRender.h`](SoundRender.h.md) · [`Sound.h`](Sound.h.md) · [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [`SoundRender_Effects.h`](SoundRender_Effects.h.md) · [`SoundRender_Scene.h`](SoundRender_Scene.h.md)
**Used by** — [`OpenALDeviceList.cpp`](OpenALDeviceList.cpp.md) · [`SoundRender_Core.cpp`](SoundRender_Core.cpp.md) · [`SoundRender_CoreA.cpp`](SoundRender_CoreA.cpp.md) · [`SoundRender_CoreA.h`](SoundRender_CoreA.h.md) · [`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) · [`SoundRender_Core_SourceManager.cpp`](SoundRender_Core_SourceManager.cpp.md) · [`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md) · [`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md) · [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) · [`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md) · [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md) · [`SoundRender_Source.cpp`](SoundRender_Source.cpp.md) · [`xr_streamsnd.cpp`](xr_streamsnd.cpp.md)
**Tier floor** — T2: a declaration surface with no layout or device concerns.

## Purpose

Declares the surface implemented across four files —
[`SoundRender_Core.cpp`](SoundRender_Core.cpp.md) (lifecycle, sources, environment),
[`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) (the per-frame passes),
[`SoundRender_Core_SourceManager.cpp`](SoundRender_Core_SourceManager.cpp.md) (the source cache) and
[`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md) (voice allocation). The split
across four files is organisational, not architectural; a rebuild may merge them.

What is load-bearing here is the *abstract* part: the four operations left for a backend to fill,
which together are the entire device seam.

## Exported units

- **`SoundCore`** — the manager. Implements the abstract manager interface from
  [`Sound.h`](Sound.h.md).
- **`initialize_devices_list`** *(abstract)* — enumerate output devices; may run before any device
  is opened, because the settings screen needs the list.
- **`initialize`** *(abstract)* — open the selected device, create the voice pool, detect the
  reverb and float-PCM capabilities.
- **`clear`** *(abstract)* — destroy voices and close the device.
- **`set_master_volume`** *(abstract)* — the single global gain.
- **`update_listener`** *(virtual)* — the core computes the reverb blend; a backend extends it to
  push the transform at the device.
- **`create_source` / `destroy_source`** — the shared-source cache, keyed by asset path.
- **`take_voice(emitter)`** — assign a voice, evicting the weakest holder. Contract in
  [`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md).
- **`may_play(emitter)`** — would a voice be available for this emitter's rank? Same file.
- **`listener_params` / `listener_position`** — read-only views used by emitters for distance.
- **`Listener.to_right_handed()`** — mirrors the Z axis of position and of all three orientation
  axes. The engine's world is left-handed and the mixer seam's is right-handed; every value
  crossing that boundary is mirrored, never rotated, so that a sound to the player's left stays on
  the left.

## Notes

`locked` is published through the manager interface so that callers can assert against re-entry.
The scene list, source map and voice list are visible to subclasses because a backend's
initialization fills the voice list directly — a rebuild with a cleaner factory boundary loses
nothing.
