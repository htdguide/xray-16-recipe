# src/xrGame/smart_cover_transition.hpp

> Declares one edge of a smart cover's transition graph: a guarded set of animations that carries a creature from one loophole to another.

**Needs** — [`smart_cover_transition.cpp`](smart_cover_transition.cpp.md) · [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md) · [`ai_monster_space.h`](ai_monster_space.h.md)
**Used by** — [`smart_cover_description.cpp`](smart_cover_description.cpp.md) · [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`smart_cover_transition.cpp`](smart_cover_transition.cpp.md) · [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_movement_manager_smart_cover_loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md)
**Tier floor** — T2: authored data plus a script-evaluated guard

## Purpose

Declares the surface implemented in
[`smart_cover_transition.cpp`](smart_cover_transition.cpp.md). A transition *action* is
the payload on an edge of the per-description transition graph: a precondition expressed
as a named script function with a parameter string, plus the animations that realize the
move.

Exported units:

- `action(table)` — build from the authored configuration table.
- `applicable()` — ask the script guard whether this edge may be taken now.
- `animation(body_state)` — the animation that leaves the creature in a given body state.
- `animation()` — any animation of this edge, chosen at random.
- `animations()` — the whole set, for callers that plan over it.

## State

```text
RECORD transition_action
  precondition_functor : text                 # name of a script predicate
  precondition_params  : text                 # single string argument handed to it
  animations           : list<animation_action>   # invariant: non-empty
```
