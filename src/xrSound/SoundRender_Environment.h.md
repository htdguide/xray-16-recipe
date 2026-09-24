# src/xrSound/SoundRender_Environment.h

> Declares a reverb preset and the library that maps level-authored names to presets.

**Needs** — [`Sound.h`](Sound.h.md)
**Used by** — [`Sound.cpp`](Sound.cpp.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Effects.h`](SoundRender_Effects.h.md) · [`SoundRender_EffectsA_EAX.cpp`](SoundRender_EffectsA_EAX.cpp.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Environment.cpp`](SoundRender_Environment.cpp.md) · [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md) · [`SoundRender_Scene.h`](SoundRender_Scene.h.md)
**Tier floor** — T2: declarations only; the frozen file format lives in the `.cpp`.

## Purpose

Declares the preset and its library, implemented in
[`SoundRender_Environment.cpp`](SoundRender_Environment.cpp.md), which carries the twelve fields,
their meanings and the frozen on-disk order.

## Exported units

- **`Environment`** — the preset. Fills the opaque environment type the game-facing interface in
  [`Sound.h`](Sound.h.md) hands around, which is how the game can carry a preset without knowing
  what reverb model it describes.
- **`set_identity` / `set_default`** — the no-reverb preset and the neutral room.
- **`clamp`** — force every field into the reverb model's legal range.
- **`lerp`** — blend two presets; how boundaries are crossed.
- **`load` / `save`** — one preset from or to a chunk of the library file.
- **`SoundEnvironmentLibrary`** — load, save, unload, lookup by name or index, append, remove. The
  mutating half exists for the editor.
