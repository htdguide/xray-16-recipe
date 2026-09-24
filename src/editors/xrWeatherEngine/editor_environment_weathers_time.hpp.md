# src/editors/xrWeatherEngine/editor_environment_weathers_time.hpp

> Declares one keyframe: the record the engine interpolates, plus everything needed to edit it live.

**Needs** — [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md) · [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md) · [`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md) · [`engine_impl.cpp`](engine_impl.cpp.md)
**Tier floor** — T1: it *is* the engine's keyframe record, which the renderer reads directly.

## Purpose

Declares the surface implemented in
[`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md). The
keyframe is the editor's central document object, and it is the engine's run-time keyframe
extended, not a copy of it — the same instance the blend reads.

## State

```text
RECORD Time EXTENDS EnvironmentDescriptorMixer
  # inherited: every interpolated field — colours, fog, rain, wind, sun direction,
  # textures, durations — plus exec_time and the identifier.
  ambient               : text     # names a record in the ambients file
  sun                   : text     # names a record in the suns file
  thunderbolt_collection : text    # names a record in the thunderbolt collections file
  owner                 : optional<Weather>   # none for the interpolated keyframe
  property_holder       : PropertyHolder
```

**Invariants** — the identifier is a time of day in `HH:MM:SS` form and `exec_time` is the
same moment in seconds from midnight. The three name fields shadow inherited resolved
references; writing one re-resolves the other.

## Exported units

- **`load` / `load_from` / `save`** — read a keyframe's fields from a configuration
  section, read them under a borrowed section name, or write them.
- **`fill`** — build the keyframe's grid page, which is the editor's entire authoring
  surface.
- **`lerp`** — the interpolation hook, overridden so the interpolated keyframe carries the
  clock and the names as well as the numbers.
- **`id`** — the keyframe's time of day.
