# src/editors/xrWeatherEngine/editor_environment_suns_manager.hpp

> Declares the owner of the sun records — the named lens-flare setups a keyframe selects.

**Needs** — [`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md) · [`editor_environment_suns_gradient.cpp`](editor_environment_suns_gradient.cpp.md) · [`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md) · [`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) · [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md)
**Tier floor** — T2: it owns the records and one configuration file.

## Purpose

Declares the surface implemented in
[`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md).

## State

```text
RECORD SunsManager
  suns        : list<Sun>
  collection  : PropertyCollection
  sun_ids     : list<text>          # cached, sorted; always carries an empty name first
  changed     : bool
  environment : EditorEnvironment   # read-only; the source of the shader name list
```

## Exported units

- **`load` / `save`** — read the suns file; write it (never called — see the implementation).
- **`fill`** — add the sun list to the grid.
- **`suns_ids`** — the sorted record names, with an empty name included.
- **`unique_id`** — the renaming rule.
- **`get_flare`** — declared and stubbed; returns nothing.
