# src/editors/xrWeatherEngine/editor_environment_ambients_effect_id.hpp

> Declares one entry in an ambient's effect list: a name chosen from the effects the model defines.

**Needs** — [`editor_environment_ambients_effect_id.cpp`](editor_environment_ambients_effect_id.cpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_ambients_effect_id.cpp`](editor_environment_ambients_effect_id.cpp.md)
**Tier floor** — T3: a name and a picker.

## Purpose

Declares the surface implemented in
[`editor_environment_ambients_effect_id.cpp`](editor_environment_ambients_effect_id.cpp.md).

## State

```text
RECORD EffectId
  id              : text                # names a record in the effects file
  effects_manager : EffectsManager      # read-only; the source of the picker's options
  property_holder : PropertyHolder
```

## Exported units

- **`fill`** — the one grid row: a name chosen from a list.
- **`id`** — the chosen name.
