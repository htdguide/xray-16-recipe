# src/editors/xrWeatherEngine/editor_environment_thunderbolts_collection.hpp

> Declares one named set of thunderbolts: what a keyframe points at when it says a storm is happening.

**Needs** — [`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md) · [`editor_environment_thunderbolts_thunderbolt_id.hpp`](editor_environment_thunderbolts_thunderbolt_id.hpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md) · [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md)
**Tier floor** — T2: it is the engine's thunderbolt collection, extended.

## Purpose

Declares the surface implemented in
[`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md).

## State

```text
RECORD Collection EXTENDS RuntimeThunderboltCollection
  id          : text                 # the configuration section name
  entries     : list<ThunderboltId>  # the editable view: names
  collection  : PropertyCollection
  property_holder : PropertyHolder
  # inherited:
  palette     : list<Thunderbolt>    # the resolved records the engine picks from
```

**Invariants** — the entries and the palette are built in parallel by the same load and
never reconciled afterwards.

## Exported units

- **`load` / `save`** — read and write one configuration section.
- **`fill`** — two grid rows: the set's name and its entries.
- **`id`** — the set's name.
