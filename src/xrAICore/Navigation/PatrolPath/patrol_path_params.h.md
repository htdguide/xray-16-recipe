# src/xrAICore/Navigation/PatrolPath/patrol_path_params.h

> Declares the handle a creature or a script holds onto a named patrol path, implemented in [`patrol_path_params.cpp`](patrol_path_params.cpp.md).

**Needs** — [`patrol_path_params.cpp`](patrol_path_params.cpp.md) · [`patrol_path.h`](patrol_path.h.md) · [`../game_graph_space.h`](../game_graph_space.h.md) · [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`patrol_path_params.cpp`](patrol_path_params.cpp.md) · [`patrol_path_params_inline.h`](patrol_path_params_inline.h.md) · [`patrol_path_params_script.cpp`](patrol_path_params_script.cpp.md) · [`script_movement_action.cpp`](../../../xrGame/script_movement_action.cpp.md) · [`script_movement_action.h`](../../../xrGame/script_movement_action.h.md) · [`script_movement_action_script.cpp`](../../../xrGame/script_movement_action_script.cpp.md)
**Tier floor** — T2: a resolved reference plus four authored settings.

## Purpose

Declares the type implemented in [`patrol_path_params.cpp`](patrol_path_params.cpp.md) and
exported to scripts by
[`patrol_path_params_script.cpp`](patrol_path_params_script.cpp.md). It is the bundle a
creature is handed when it is told to patrol: *which* path, *where* on it to start, what to do
when it ends, and whether to walk it in order or at random.

## Exported units

- construction from a path name plus the four settings, all of which have defaults
- `count()` — how many waypoints the path has
- `point(index)` / `name(index)` — a waypoint's position and its authored name
- `level_vertex_id(index)` / `game_vertex_id(index)` — the waypoint's mesh and cross-level vertices
- `point(name)` — the index of the waypoint with that name
- `point(position)` — the index of the waypoint nearest a position
- `flag(index, bit)` / `flags(index)` — the authored flag bits on a waypoint
- `terminal(index)` — has this waypoint no outgoing links
