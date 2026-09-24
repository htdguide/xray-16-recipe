# src/xrGame/object_handler_planner.h

> Declares the object-handling planner — one goal-directed search per creature over what its hands are doing — implemented in [`object_handler_planner.cpp`](object_handler_planner.cpp.md) and its two item-kind siblings.

**Needs** — [`action_planner.h`](action_planner.h.md) · [`object_handler_planner_inline.h`](object_handler_planner_inline.h.md) · [`object_handler_space.h`](object_handler_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`ai_stalker.cpp`](ai/stalker/ai_stalker.cpp.md) · [`ai_stalker_debug.cpp`](ai/stalker/ai_stalker_debug.cpp.md) · [`ai_stalker_fire.cpp`](ai/stalker/ai_stalker_fire.cpp.md) · [`ai_stalker_script_entity.cpp`](ai/stalker/ai_stalker_script_entity.cpp.md) · [`object_actions.cpp`](object_actions.cpp.md) · [`object_handler.cpp`](object_handler.cpp.md) · [`object_handler_planner.cpp`](object_handler_planner.cpp.md) · [`object_handler_planner_impl.h`](object_handler_planner_impl.h.md) · [`object_handler_planner_inline.h`](object_handler_planner_inline.h.md) · [`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) · [`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`stalker_animation_torso.cpp`](stalker_animation_torso.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CObjectHandlerPlanner`, a specialization of the engine's generic action planner
whose world is "what the creature is doing with the item in its hands". Substance is split:
[`object_handler_planner.cpp`](object_handler_planner.cpp.md) (goals, item set, lifecycle),
[`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md) (the operator and
evaluator set for a firearm) and
[`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) (the same for a
thrown object). The property-identifier arithmetic is in
[`object_handler_planner_impl.h`](object_handler_planner_impl.h.md).

Exported units:

- `CObjectHandlerPlanner` — the planner. Also holds the burst schedule: minimum and maximum
  burst size and interval, the currently rolled values, and when to roll again.
- `setup` — bind to a creature, clear everything, install the no-item operator, aim at idle.
- `update` — step the planner.
- `add_item` / `remove_item` — grow and shrink the planner's operator and evaluator sets as
  the inventory changes.
- `set_goal` — translate a named object action plus a target object into a target world
  state, and re-roll the burst schedule.
- `uid` — combine an entity identifier and a property or operator name into one identifier.
- `object_action` / `action_object_id` / `action_state_id` and the `current_` forms — take
  that identifier apart again.
- `add_condition` / `add_effect` — attach a per-item precondition or effect to an operator.
- `add_evaluators` / `add_operators` / `remove_evaluators` / `remove_operators` — the per-item
  set construction, one overload per item kind.
- `object_property` — map an object action onto the world property that satisfies it.
- `action2string` / `property2string` — human-readable names, in logging builds only.
