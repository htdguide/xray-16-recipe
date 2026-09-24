# src/xrGame/alife_story_registry.h

> Declares the story-identifier index, implemented in [`alife_story_registry.cpp`](alife_story_registry.cpp.md) and [`alife_story_registry_inline.h`](alife_story_registry_inline.h.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`alife_story_registry_inline.h`](alife_story_registry_inline.h.md)
**Used by** — [`GameTask.cpp`](GameTask.cpp.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md) · [`alife_story_registry.cpp`](alife_story_registry.cpp.md) · [`alife_story_registry_inline.h`](alife_story_registry_inline.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeStoryRegistry`: a map from authored story identifier to server object.
Insertion is in [`alife_story_registry.cpp`](alife_story_registry.cpp.md); lookup and
removal are in
[`alife_story_registry_inline.h`](alife_story_registry_inline.h.md).

Like the smart-terrain index it **owns nothing**; it is a view onto entities the object
registry owns. And like that index it is offered every entity and filters, rather than
being told only about the ones that qualify — which keeps the qualification rule in one
place.

Exported units:

- `CALifeStoryRegistry` — a map, default-constructed.
- `add` / `remove` — offered every entity; keeps those with a story identifier.
- `object` — resolve a story identifier; absence is a diagnosed fault unless tolerated.
- `objects` — the whole index.
