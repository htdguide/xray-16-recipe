# src/xrGame/smart_cover_loophole.h

> Declares one firing position inside a smart cover: where it looks, how far and how wide, what a creature can do there, and how to get from one action to another.

**Needs** — [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`smart_cover_action.h`](smart_cover_action.h.md) · [`xrAICore/Navigation/graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`cover_evaluators.cpp`](cover_evaluators.cpp.md) · [`cover_manager.cpp`](cover_manager.cpp.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_description.cpp`](smart_cover_description.cpp.md) · [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) · [`smart_cover_loophole.cpp`](smart_cover_loophole.cpp.md) · [`smart_cover_loophole_inline.h`](smart_cover_loophole_inline.h.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`smart_cover_object.cpp`](smart_cover_object.cpp.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md) · [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_movement_manager_smart_cover_loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md)
**Tier floor** — T2: a parsed record with an inner graph

## Purpose

Declares the surface implemented in
[`smart_cover_loophole.cpp`](smart_cover_loophole.cpp.md) and
[`smart_cover_loophole_inline.h`](smart_cover_loophole_inline.h.md).

A loophole is a *place to stand and look* inside a cover. Everything geometric about it is
in the cover object's local frame, so one authored description can be placed anywhere on a
level; [`smart_cover.cpp`](smart_cover.cpp.md) transforms it to world space on demand.

Note the two levels of graph. The *description* owns a graph whose vertices are loopholes.
Each *loophole* owns a second graph whose vertices are its own actions. A creature's plan
through a cover is a path in the first, and each step of it a path in the second.

## Exported units

- **The class** — geometry, actions, an inner transition graph, and three flags.
- **`id`** — the authored name; the vertex name in the description's graph.
- **`fov_position`, `fov_direction`, `danger_fov_direction`, `enter_direction`** — local
  geometry: where the creature's eye sits, where the loophole looks, the narrower arc that
  counts as threatening, and which way a creature faces coming in.
- **`fov`, `danger_fov`, `range`** — the wide arc, the narrow arc, and the reach.
- **`actions`** — what a creature can do here, keyed by name.
- **`is_action_available`** — whether a named action exists here.
- **`action_animations`** — the clip list for a purpose of a named action.
- **`transition_animations`** — the clips that carry a creature from one action to another
  within this loophole.
- **`usable`** — derived: false when no action was authored.
- **`enterable` / `exitable`** — derived by the description from its own graph; settable
  for that reason.
- **`exit_position`** — the local spot the exit action places the creature at, if there is
  one.
