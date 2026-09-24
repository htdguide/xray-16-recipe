# src/xrGame/alife_monster_detail_path_manager.h

> Declares the offline mover that walks an alife creature along a game-graph path, implemented in [`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [`alife_monster_detail_path_manager_inline.h`](alife_monster_detail_path_manager_inline.h.md)
**Used by** — [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md) · [`alife_monster_detail_path_manager_inline.h`](alife_monster_detail_path_manager_inline.h.md) · [`alife_monster_detail_path_manager_script.cpp`](alife_monster_detail_path_manager_script.cpp.md) · [`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md) · [`alife_monster_movement_manager_script.cpp`](alife_monster_movement_manager_script.cpp.md) · [`alife_online_offline_group.cpp`](alife_online_offline_group.cpp.md) · [`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md)
**Tier floor** — T2: a plain record plus a path buffer; nothing device- or layout-facing

## Purpose

Declares `CALifeMonsterDetailPathManager`, the component an offline creature's movement
manager owns to answer one question per alife tick: *where am I now, given that I am
walking to there*. The substance — the search, the distance-per-tick integration and the
online/offline handover — is in
[`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md).

The header is worth reading for two shape decisions it fixes.

First, a **destination is a triple**, not a point: a game-graph vertex, a level-graph
vertex, and a precise position. Offline movement happens on the game graph; the two finer
fields matter only when the destination lies on the currently loaded level, and are
carried along so that a creature switching online lands on a legal navigation vertex
rather than an arbitrary coordinate.

Second, the **stored path is inverted** — the destination vertex is first and the current
vertex is last. Following the path consumes from the end. The original notes this is
because removing from the end of a growable array is cheap; a rebuild may store it either
way, but must preserve that following is destructive (the path is consumed, and its
emptiness is the definition of *failed*).

Exported units:

- `CALifeMonsterDetailPathManager` — constructed against the owning movement-manager
  holder; the destination is seeded from the owner's current position, so a
  freshly-built manager is already "arrived".
- `target(game vertex, level vertex, position)` / `target(game vertex)` /
  `target(smart-terrain task)` — set the destination; the one-argument form fills the
  finer two fields from the game graph vertex's own level point.
- `update()` — advance by the game time elapsed since the last advance.
- `on_switch_online` / `on_switch_offline` — the handover hooks.
- `speed` (get and set) — travel speed in graph-distance units per unit of game time.
- `completed` — destination reached; `actual` — the cached path still leads to the
  current destination; `failed` — no path exists; `make_inactual` — discard the path so
  the next advance re-searches.
- `path`, `walked_distance`, `draw_level_position` — read-only views, the last for debug
  rendering.
- A script registration hook; the exported surface is in
  [`alife_monster_detail_path_manager_script.cpp`](alife_monster_detail_path_manager_script.cpp.md).
