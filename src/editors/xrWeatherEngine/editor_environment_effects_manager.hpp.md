# src/editors/xrWeatherEngine/editor_environment_effects_manager.hpp

> Declares the owner of the effect records — the one-shot particle-and-sound bursts an ambient fires occasionally.

**Needs** — [`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_ambients_effect_id.cpp`](editor_environment_ambients_effect_id.cpp.md) · [`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md) · [`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md)
**Tier floor** — T2: it owns the records and one configuration file.

## Purpose

Declares the surface implemented in
[`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md).

## State

```text
RECORD EffectsManager
  effects     : list<Effect>
  collection  : PropertyCollection
  effect_ids  : list<text>        # cached, sorted; rebuilt when changed is set
  changed     : bool
  environment : EditorEnvironment # for the particle-system name list an effect browses
```

## Exported units

- **`load` / `save`** — read and write the effects file.
- **`fill`** — add the effect list to the grid, under the ambients group.
- **`effects_ids`** — the sorted record names, for an ambient's effect picker.
- **`unique_id`** — the renaming rule.
- **`environment`** — the weather system, so an effect can browse particle-system names.
