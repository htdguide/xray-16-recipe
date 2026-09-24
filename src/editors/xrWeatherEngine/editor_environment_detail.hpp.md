# src/editors/xrWeatherEngine/editor_environment_detail.hpp

> Declares the two helpers every part of the weather model needs: a human-friendly sort order, and a real path for a file dialog.

**Needs** — [`editor_environment_detail.cpp`](editor_environment_detail.cpp.md)
**Used by** — [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md) · [`editor_environment_detail.cpp`](editor_environment_detail.cpp.md) · [`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md) · [`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_sound_channels_manager.cpp`](editor_environment_sound_channels_manager.cpp.md) · [`editor_environment_sound_channels_source.cpp`](editor_environment_sound_channels_source.cpp.md) · [`editor_environment_suns_flare.cpp`](editor_environment_suns_flare.cpp.md) · [`editor_environment_suns_gradient.cpp`](editor_environment_suns_gradient.cpp.md) · [`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md) · [`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) · [`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md) · [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · [`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md) · _and 2 more_
**Tier floor** — T2: a comparator and a path resolution.

## Purpose

Declares the surface implemented in
[`editor_environment_detail.cpp`](editor_environment_detail.cpp.md).

## State

`Stateless.`

## Exported units

- **the natural-order comparator** — orders two names the way a person reads them, for
  plain text and for interned strings alike.
- **`real_path`** — turn a virtual filesystem folder and a relative path into a location
  a native file dialog can open.
