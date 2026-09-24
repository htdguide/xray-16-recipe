# src/xrGame/level_sounds.h

> Declares the level's ambient soundscape: authored looping and intermittent point sources, and the time-of-day music playlist.

**Needs** — [`level_sounds.cpp`](level_sounds.cpp.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`Level.cpp`](Level.cpp.md) · [`Level_load.cpp`](Level_load.cpp.md) · [`level_sounds.cpp`](level_sounds.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the two ambient sound records and the manager that owns them. See
[`level_sounds.cpp`](level_sounds.cpp.md).

Exported units:

- `SStaticSound` — one authored point source: its sound, position, volume, pitch, and three
  time windows (when it may sound at all, how long a burst lasts, how long between bursts),
  plus the two clock deadlines that drive it.
- `SStaticSound::Load` · `SStaticSound::Update` — read one from the level's sound chunk;
  advance its play state for this frame.
- `SMusicTrack` — one playlist entry: up to three sound sources (a stereo one, or a left and
  a right), its active window, its pause range and its volume.
- `SMusicTrack::Load` · `in` · `IsPlaying` · `Play` · `Stop` · `SetVolume`.
- `CLevelSoundManager` — the two collections, the currently playing track index and the time
  the next selection may happen.
- `CLevelSoundManager::Load` · `Unload` · `Update` — bound to the level's own load, unload
  and frame.
