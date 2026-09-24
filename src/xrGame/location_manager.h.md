# src/xrGame/location_manager.h

> Declares a creature's terrain preference: which kinds of game-graph terrain it is willing to travel through, and how strongly.

**Needs** — [`location_manager_inline.h`](location_manager_inline.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ai_rat_templates.cpp`](ai/monsters/rats/ai_rat_templates.cpp.md) · [`game_location_selector.h`](game_location_selector.h.md) · [`game_location_selector_inline.h`](game_location_selector_inline.h.md) · [`location_manager.cpp`](location_manager.cpp.md) · [`location_manager_inline.h`](location_manager_inline.h.md) · [`map_location.cpp`](map_location.cpp.md) · [`movement_manager.cpp`](movement_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CLocationManager`. See [`location_manager.cpp`](location_manager.cpp.md).

Exported units:

- `CLocationManager` — the owning object and its list of terrain masks.
- `Load` — read the preference from a configuration section.
- `reload` — override it from the entity's own spawn-time configuration.
- `clear_location_types` · `add_location_type` — build the list by hand, one mask at a time.
- `vertex_types` — the list, in [`location_manager_inline.h`](location_manager_inline.h.md).
