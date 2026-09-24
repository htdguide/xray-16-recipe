# src/xrGame/alife_smart_terrain_task.h

> Declares the job destination, implemented in [`alife_smart_terrain_task.cpp`](alife_smart_terrain_task.cpp.md).

**Needs** — [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`alife_smart_terrain_task_inline.h`](alife_smart_terrain_task_inline.h.md)
**Used by** — [`monster_state_smart_terrain_task.h`](ai/monsters/states/monster_state_smart_terrain_task.h.md) · [`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md) · [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`alife_monster_detail_path_manager_script.cpp`](alife_monster_detail_path_manager_script.cpp.md) · [`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md) · [`alife_smart_terrain_task.cpp`](alife_smart_terrain_task.cpp.md) · [`alife_smart_terrain_task_inline.h`](alife_smart_terrain_task_inline.h.md) · [`alife_smart_terrain_task_script.cpp`](alife_smart_terrain_task_script.cpp.md) · [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeSmartTerrainTask`, a small value type naming a place. Substance is in
[`alife_smart_terrain_task.cpp`](alife_smart_terrain_task.cpp.md); construction is in
[`alife_smart_terrain_task_inline.h`](alife_smart_terrain_task_inline.h.md).

Two of the record's fields — the patrol path's name and the point index — exist only in a
diagnostic build, where they turn a failed vertex check into a message naming the path and
point that produced it. That is a genuine trade and worth reproducing: the task is
resolved lazily, so the place a bad value *comes from* is no longer on the call stack when
it is detected.

Exported units:

- `CALifeSmartTerrainTask` — five constructors: a patrol path by name (point zero
  implied), a path and a point index, both again with a shared string, and a game-vertex
  plus level-vertex pair.
- `game_vertex_id`, `level_vertex_id`, `position` — the destination triple.
- A script registration hook; surface in
  [`alife_smart_terrain_task_script.cpp`](alife_smart_terrain_task_script.cpp.md).
