# src/xrGame/level_path_manager.h

> Declares the level-graph flavour of the path manager: a cached route across one level's navigation mesh, aware of the searcher's restrictors.

**Needs** — [`abstract_path_manager.h`](abstract_path_manager.h.md) · [`level_path_manager_inline.h`](level_path_manager_inline.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`control_path_builder.cpp`](ai/monsters/control_path_builder.cpp.md) · [`control_path_builder_base.cpp`](ai/monsters/control_path_builder_base.cpp.md) · [`control_path_builder_base_path.cpp`](ai/monsters/control_path_builder_base_path.cpp.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`level_path_manager_inline.h`](level_path_manager_inline.h.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md) · [`movement_manager_level.cpp`](movement_manager_level.cpp.md) · [`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md) · [`script_entity.cpp`](script_entity.cpp.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md) · [`stalker_combat_actions.cpp`](stalker_combat_actions.cpp.md) · [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) · [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md) · _and 2 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the level-graph specialization of the generic path manager. See
[`level_path_manager_inline.h`](level_path_manager_inline.h.md) for the substance.

Exported units:

- the level path manager, parameterized by the vertex scoring function and the identifier
  widths, and sealed against further derivation.
- `reinit` — rebind to a graph and drop the cached route.
- `actual` — whether the cached route still runs from where the creature is to where it
  wants to go.
- `build_path` — run the search, with validation and a diagnostic on failure.
- `on_restrictions_change` — forget the remembered failure.
- `before_search` · `after_search` · `check_vertex` — the restrictor-aware hooks into the
  generic search.

**Notes** — the movement manager and the level path builder are named as privileged callers
so they can reach the protected hooks. That is a C++ access arrangement; what it records is
that this manager is not a general-purpose service — it is a part of the movement pipeline
and only the pipeline drives it.
