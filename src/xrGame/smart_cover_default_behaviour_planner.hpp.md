# src/xrGame/smart_cover_default_behaviour_planner.hpp

> Declares the small planner that decides what a creature in a cover should be doing when it has no enemy: idle, or look out.

**Needs** — [`action_planner_action.h`](action_planner_action.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md)
**Used by** — [`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md) · [`smart_cover_default_behaviour_planner_inline.hpp`](smart_cover_default_behaviour_planner_inline.hpp.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md)
**Tier floor** — T2: a two-operator plan search per cycle

## Purpose

Declares the surface implemented in
[`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md)
and
[`smart_cover_default_behaviour_planner_inline.hpp`](smart_cover_default_behaviour_planner_inline.hpp.md).

This is a planner that is an *action of another planner*: the target selector
([`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md))
offers it as one of its options, applicable when the creature is in default or combat cover
behaviour. Nesting the peaceful case in its own planner keeps the combat operators in the
selector free of preconditions about not having an enemy.

## Exported units

- **The class** — a nested planner over the animation planner.
- **`setup`** — register its two evaluators and two operators and set its goal.
- **`initialize` / `update` / `finalize`** — pure delegation to the base planner.
- **`idle_time` / `lookout_time`** — two stored intervals; see the notes.
- **`object_name`** — diagnostic identity.

## Notes

The two stored intervals are written and read by nothing. The dwell timing that actually
runs lives on the animation planner
([`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md)), where both
dwell evaluators share it. A rebuild should drop these two fields.
