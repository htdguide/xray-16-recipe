# src/xrGame/movement_manager_space.h

> The four kinds of path a creature can be asked to follow.

**Needs** — _(none)_
**Used by** — [`control_path_builder_base.h`](ai/monsters/control_path_builder_base.h.md) · [`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md) · [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) · [`alife_monster_movement_manager_script.cpp`](alife_monster_movement_manager_script.cpp.md) · [`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager.h`](movement_manager.h.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) · [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md) · [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md) · [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`stalker_search_actions.cpp`](stalker_search_actions.cpp.md)
**Tier floor** — T4: an enumeration

## Purpose

One enumeration in its own file so that the movement manager, the script binding, the
stalker brain and every action that commands motion can name a path kind without including
each other. That is the only reason for the file.

## State

`Stateless.`

## The path type

```text
ENUM PathType
  game_path     # destination is a vertex of the coarse cross-level graph;
                # resolves down through a level path and then a detail path
  level_path    # destination is a vertex of this level's navigation mesh
  patrol_path   # destination is chosen, point by point, from an authored patrol route
  no_path       # movement manager idles; the creature is positioned by something else
  dummy         # the all-ones sentinel for "not set"
```

**Invariants** — the sentinel is the all-ones pattern of a 32-bit unsigned word, so the
storage width is load-bearing: the value is saved and exposed to scripts at that width.
The three real kinds are a *containment* hierarchy, not alternatives: a game path is
decomposed into level paths, and a level path into a detail path. Choosing `game_path`
therefore runs strictly more pipeline stages than `level_path`, and a rebuild must run
them in the order named.
