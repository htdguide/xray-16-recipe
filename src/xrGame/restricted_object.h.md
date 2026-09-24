# src/xrGame/restricted_object.h

> Declares the per-creature view of the restrictor system: which restrictors constrain me, is a place accessible, and a temporary border for the duration of one path.

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`restricted_object_inline.h`](restricted_object_inline.h.md)
**Used by** — [`abstract_location_selector.h`](abstract_location_selector.h.md) · [`abstract_location_selector_inline.h`](abstract_location_selector_inline.h.md) · [`abstract_path_manager.h`](abstract_path_manager.h.md) · [`abstract_path_manager_inline.h`](abstract_path_manager_inline.h.md) · [`anomaly_detector.cpp`](ai/monsters/anomaly_detector.cpp.md) · [`monster_home.cpp`](ai/monsters/monster_home.cpp.md) · [`monster_state_squad_rest_follow_inline.h`](ai/monsters/states/monster_state_squad_rest_follow_inline.h.md) · [`monster_state_squad_rest_inline.h`](ai/monsters/states/monster_state_squad_rest_inline.h.md) · [`cover_evaluators.cpp`](cover_evaluators.cpp.md) · [`cover_evaluators.h`](cover_evaluators.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`detail_path_manager_smooth.cpp`](detail_path_manager_smooth.cpp.md) · [`item_manager.cpp`](item_manager.cpp.md) · [`level_location_selector_inline.h`](level_location_selector_inline.h.md) · _and 6 more_
**Tier floor** — T2: a mixin over a creature; per-query work goes to the restriction manager

## Purpose

Declares the surface every creature uses to ask where it is allowed to be. The restriction
*sets* live in the level's restriction manager, keyed by entity identifier; this type is the
per-creature facade over them plus the two pieces of state the creature itself must own — the
temporary path border, and whether the pathfinder's cached cost model is still valid.

The substance is in [`restricted_object.cpp`](restricted_object.cpp.md). The obstacle-aware
specialization is [`restricted_object_obstacle.h`](restricted_object_obstacle.h.md).

Exported units:

- `net_Spawn` / `net_Destroy` — build this creature's restriction set from its server record
  at spawn, and drop it at destruction.
- `add_border` (three forms) / `remove_border` — a temporary extra restriction confining a
  path in progress. Declared virtual so the obstacle variant can widen them.
- `accessible` (four forms) / `accessible_nearest` — the queries the pathfinder and the
  behaviour layer ask.
- `add_restrictions` / `remove_restrictions` (identifier-list and name-string forms) —
  change this creature's set at run time.
- `remove_all_restrictions` — clear it, by type or entirely.
- `in_restrictions` / `out_restrictions` / `base_in_restrictions` / `base_out_restrictions` —
  read the current sets and the spawn-time ones as name strings.
- `applied` / `actual` — has a border been installed, and is the cached cost model current.
