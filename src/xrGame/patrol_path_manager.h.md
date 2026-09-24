# src/xrGame/patrol_path_manager.h

> Declares the walker of authored patrol routes, implemented in [`patrol_path_manager.cpp`](patrol_path_manager.cpp.md).

**Needs** — [`patrol_path_manager_inline.h`](patrol_path_manager_inline.h.md) · [`Level.h`](Level.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_storage.h`](../xrAICore/Navigation/PatrolPath/patrol_path_storage.h.md) · [`xrScriptEngine/script_callback_ex.h`](../xrScriptEngine/script_callback_ex.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`base_monster_script.cpp`](ai/monsters/basemonster/base_monster_script.cpp.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md) · [`patrol_path_manager.cpp`](patrol_path_manager.cpp.md) · [`patrol_path_manager_inline.h`](patrol_path_manager_inline.h.md) · [`script_entity.cpp`](script_entity.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CPatrolPathManager`, the component that walks a creature along an authored patrol
route: a graph of named waypoints with weighted edges, shipped with the level. Substance is in
[`patrol_path_manager.cpp`](patrol_path_manager.cpp.md); the state setters, which carry the
invalidation rules, are in
[`patrol_path_manager_inline.h`](patrol_path_manager_inline.h.md).

Exported units:

- `CPatrolPathManager` — holds the route, where on it the creature is and was, the start and
  route policies, the randomness flag, and a script callback.
- `set_path` (three forms) — adopt a route by object or by name, optionally with both policies
  and the randomness flag in one call.
- `set_start_type` / `set_route_type` / `set_random` — the policies individually.
- `set_previous_point` / `set_start_point` — place the creature on the route explicitly.
- `select_point` — the one real operation: choose the next waypoint and report its navigation
  vertex.
- `destination_position` / `get_current_point_index` / `path_name` — what was chosen.
- `actual` / `completed` / `failed` / `make_inactual` — the freshness and termination flags.
- `extrapolate_path` / `extrapolate_callback` — whether the creature may walk past the
  waypoint, answered by a script hook.
- `reinit` — return to the unbound state.
