# src/xrGame/alife_monster_patrol_path_manager.h

> Declares the offline patrol-path cursor, implemented in [`alife_monster_patrol_path_manager.cpp`](alife_monster_patrol_path_manager.cpp.md).

**Needs** — [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`alife_monster_patrol_path_manager_inline.h`](alife_monster_patrol_path_manager_inline.h.md)
**Used by** — [`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md) · [`alife_monster_movement_manager_script.cpp`](alife_monster_movement_manager_script.cpp.md) · [`alife_monster_patrol_path_manager.cpp`](alife_monster_patrol_path_manager.cpp.md) · [`alife_monster_patrol_path_manager_inline.h`](alife_monster_patrol_path_manager_inline.h.md) · [`alife_monster_patrol_path_manager_script.cpp`](alife_monster_patrol_path_manager_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeMonsterPatrolPathManager`. Substance is in
[`alife_monster_patrol_path_manager.cpp`](alife_monster_patrol_path_manager.cpp.md).

The two enumerations it depends on — patrol *start type* and patrol *route type* — are
defined with the patrol path data structure rather than here, because they are properties
of how a path may be traversed and are shared with the online movement manager.

Exported units:

- `CALifeMonsterPatrolPathManager` — constructed against the owning movement-manager
  holder.
- `path` — install a path, by shared name, by raw name, or by direct reference.
- `start_type`, `route_type`, `use_randomness`, `start_vertex_index` — the four
  traversal settings, get and set.
- `update` — the per-tick cursor advance.
- `actual`, `completed` — cursor status.
- `target_game_vertex_id`, `target_level_vertex_id`, `target_position` — the current
  destination, as the triple the detail mover wants.
- A script registration hook; surface in
  [`alife_monster_patrol_path_manager_script.cpp`](alife_monster_patrol_path_manager_script.cpp.md).
