# src/editors/xrWeatherEngine/editor_environment_thunderbolts_thunderbolt_id.hpp

> Declares one member of a thunderbolt set: a name chosen from the thunderbolts the model defines.

**Needs** — [`editor_environment_thunderbolts_thunderbolt_id.cpp`](editor_environment_thunderbolts_thunderbolt_id.cpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md) · [`editor_environment_thunderbolts_collection.hpp`](editor_environment_thunderbolts_collection.hpp.md) · [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · [`editor_environment_thunderbolts_thunderbolt_id.cpp`](editor_environment_thunderbolts_thunderbolt_id.cpp.md)
**Tier floor** — T3: a name and a picker.

## Purpose

Declares the surface implemented in
[`editor_environment_thunderbolts_thunderbolt_id.cpp`](editor_environment_thunderbolts_thunderbolt_id.cpp.md).
The third instance of the same reference-with-a-picker shape as
[`effect_id`](editor_environment_ambients_effect_id.hpp.md) and
[`sound_id`](editor_environment_ambients_sound_id.hpp.md).

## State

```text
RECORD ThunderboltId
  id      : text                  # names a record in the thunderbolts file
  manager : ThunderboltsManager   # read-only; the source of the picker's options
  property_holder : PropertyHolder
```

## Exported units

- **`fill`** — the one grid row: a name chosen from a list.
- **`id`** — the chosen name.
