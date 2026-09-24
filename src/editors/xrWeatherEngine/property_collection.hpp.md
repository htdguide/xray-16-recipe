# src/editors/xrWeatherEngine/property_collection.hpp

> The adapter that makes any list of editable objects look to the property grid like an add/remove/reorder collection.

**Needs** — [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`property_collection_inline.hpp`](property_collection_inline.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Common/object_broker.h`](../../Common/object_broker.h.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md) · [`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md) · [`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md) · [`editor_environment_sound_channels_manager.cpp`](editor_environment_sound_channels_manager.cpp.md) · [`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md) · [`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md) · [`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md) · [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · [`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md) · [`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`property_collection_inline.hpp`](property_collection_inline.hpp.md)
**Tier floor** — T2: it owns the lifetime of the elements it holds.

## Purpose

Declares the surface implemented in [`property_collection_inline.hpp`](property_collection_inline.hpp.md).
Everything in the weather model that an author can add to or remove from — weather cycles,
keyframes, ambients, effects, sound channels, sound sources, suns, thunderbolts,
thunderbolt collections, and the identifier lists that reference them — is one of these,
parameterised over the element list and over the object that owns it.

## State

```text
RECORD PropertyCollection
  elements    : list<Element>       # borrowed: the owner declares it, this adapts it
  owner       : Owner               # the object the elements belong to
  changed_flag : optional<bool>     # borrowed; set on every mutation, cleared by the owner
```

**Invariants** — the adapter mutates the owner's list in place and destroys elements the
list drops, so no other code may remove an element from that list. Setting `changed_flag`
is how the owner's cached, sorted identifier list learns it is stale.

## Exported units

- **construct** — bind to an owner's element list and optional change flag.
- **`owner`** — the object the elements belong to, so element factories can reach it.
- **`clear` / `size` / `item` / `index`** — the grid's read surface over the list.
- **`insert` / `erase` / `destroy`** — the grid's mutation surface; `erase` and `destroy`
  release the element.
- **`display_name`** — the label the grid shows for one element. Declared here, defined
  per element type by its owner's file.
- **`create`** — make a new element for the grid's "add" command. Declared here, defined
  per element type by its owner's file.
- **`unique_id` / `generate_unique_id`** — the naming rule for new and renamed elements.
