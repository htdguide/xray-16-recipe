# src/editors/xrWeatherEngine/editor_environment_ambients_manager.hpp

> Declares the owner of the ambient records — the named bundles of background sound and occasional effects a keyframe points at.

**Needs** — [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md)
**Tier floor** — T2: it owns the ambient records and one configuration file.

## Purpose

Declares the surface implemented in
[`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md).

## State

```text
RECORD AmbientsManager
  ambients     : list<Ambient>
  collection   : PropertyCollection
  ambient_ids  : list<text>          # cached, sorted; rebuilt when changed is set
  changed      : bool
  environment  : EditorEnvironment   # read-only; used to reach the two sibling managers
```

## Exported units

- **`load` / `save`** — read every ambient record from the shared configuration, write them
  all back to the ambients file.
- **`fill`** — add the ambient list to the grid.
- **`ambients_ids`** — the sorted record names, for the keyframe's ambient picker.
- **`unique_id`** — the renaming rule.
- **`get_ambient`** — resolve a name to a record; fails hard on an unknown name.
- **`effects_manager` / `sounds_manager`** — the two sibling managers an ambient's children
  browse.
