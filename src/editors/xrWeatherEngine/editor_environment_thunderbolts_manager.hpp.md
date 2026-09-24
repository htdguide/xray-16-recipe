# src/editors/xrWeatherEngine/editor_environment_thunderbolts_manager.hpp

> Declares the owner of two related files — the individual thunderbolts and the named sets a keyframe draws from — plus the storm parameters shared by the whole world.

**Needs** — [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md) · [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · [`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md) · [`editor_environment_thunderbolts_thunderbolt_id.cpp`](editor_environment_thunderbolts_thunderbolt_id.cpp.md) · [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md)
**Tier floor** — T2: it owns two record sets and three configuration files.

## Purpose

Declares the surface implemented in
[`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md).
The only sub-manager with **two** editable lists, because thunderbolts are authored
individually and then grouped into the sets a keyframe names.

## State

```text
RECORD ThunderboltsManager
  thunderbolts        : list<Thunderbolt>
  thunderbolt_collection : PropertyCollection
  thunderbolts_changed   : bool

  collections         : list<Collection>
  collections_collection : PropertyCollection
  collections_changed    : bool

  thunderbolt_ids     : list<text>    # cached, sorted
  collection_ids      : list<text>    # cached, sorted; carries an empty name first
  property_holder     : PropertyHolder
  environment         : EditorEnvironment   # the storm parameters live on it, not here
```

## Exported units

- **`load` / `save`** — read both files; write both, plus the world's storm parameters.
- **`fill`** — the eight storm parameters and the two lists.
- **`thunderbolts_ids` / `collections_ids`** — the two sorted name lists.
- **`unique_thunderbolt_id` / `unique_collection_id`** — the two renaming rules.
- **`description` / `get_collection`** — resolve a name in either set.
- **`environment`** — the weather system.
