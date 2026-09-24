# src/xrEngine/LightAnimLibrary.h

> Declares the colour-animation library and the single named animation it is made of.

**Needs** — [`LightAnimLibrary.cpp`](LightAnimLibrary.cpp.md)
**Used by** — [`editor_environment_manager.cpp`](../editors/xrWeatherEngine/editor_environment_manager.cpp.md) · [`LightAnimLibrary.cpp`](LightAnimLibrary.cpp.md) · [`thunderbolt.cpp`](thunderbolt.cpp.md) · [`x_ray.cpp`](x_ray.cpp.md) · [`CustomZone.cpp`](../xrGame/CustomZone.cpp.md) · [`HangingLamp.cpp`](../xrGame/HangingLamp.cpp.md) · [`Helicopter.cpp`](../xrGame/Helicopter.cpp.md) · [`Helicopter2.cpp`](../xrGame/Helicopter2.cpp.md) · [`HitMarker.cpp`](../xrGame/HitMarker.cpp.md) · [`SimpleDetector.cpp`](../xrGame/SimpleDetector.cpp.md) · [`Torch.cpp`](../xrGame/Torch.cpp.md) · [`ZoneCampfire.cpp`](../xrGame/ZoneCampfire.cpp.md) · [`flare.cpp`](../xrGame/flare.cpp.md) · [`searchlight.cpp`](../xrGame/searchlight.cpp.md) · _and 7 more_
**Tier floor** — T2: a keyframe map and a lookup table; substance is in the implementation

## Purpose

Declares the surface implemented in [`LightAnimLibrary.cpp`](LightAnimLibrary.cpp.md): a
process-wide library of named colour animations loaded once from game data, and the
animation record itself.

Exported units:

- `CLAItem` — one named colour animation: a frame rate, a length in frames, and a sparse
  map from frame index to packed colour. Loads and saves itself from the chunked
  container; evaluates a colour at a frame or at a time, in either channel order; and
  offers the key-editing operations the weather/level editors need (insert, delete, move,
  resize, find previous/next key).
- `ELightAnimLibrary` — the library: a list of items, lookup by name, append, and the
  load/save/reload/unload cycle against the single library file.
- `LALib` — the one process-wide instance.
