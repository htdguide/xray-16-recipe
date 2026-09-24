# src/xrGame/smart_cover_evaluators.h

> Declares the thirteen world-state questions the smart-cover planners reason over: am I in a cover, is it the one I want, can I leave, has the idle run long enough.

**Needs** — [`property_evaluator.h`](property_evaluator.h.md) · [`wrapper_abstract.h`](wrapper_abstract.h.md)
**Used by** — [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md) · [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md) · [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md)
**Tier floor** — T2: planner evaluators queried every planning cycle

## Purpose

Declares the surface implemented in
[`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md).

An [evaluator](../../GLOSSARY.md) answers one question about the current world state; the
planner's operators express their preconditions and effects over those answers. This file
declares every question specific to being inside a smart cover. Reading the list is the
fastest way to understand what the smart-cover planners can reason about at all — which is
why the file is worth its own twin despite having no algorithm in it.

The evaluators split into two families by *what they are attached to*. Some are attached to
the creature and ask about its movement state; the rest are attached to the animation
planner ([`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md)) and ask
about the plan's own bookkeeping. The distinction is not cosmetic: the second family can
*write*, and two of them do.

## Exported units

Attached to the creature:

- **`in_cover_evaluator`** — is the creature currently in a smart cover.
- **`cover_entered_evaluator`** — the same question under a second name; see the notes.
- **`cover_actual_evaluator`** — is the cover the creature is in the cover it wants.
- **`loophole_actual_evaluator`** — are both the cover *and* the loophole the wanted ones.
- **`loophole_exitable_evaluator`** — does the current loophole have a way out.
- **`can_exit_loophole_with_animation`** — does the pending transition have a clip to play.

Attached to the animation planner:

- **`is_action_available_evaluator`** — does the current loophole offer a named action.
- **`loophole_hit_long_ago_evaluator`** — has enough time passed since the creature was
  hit here.
- **`default_behaviour_evaluator`** — is the creature in default or combat cover behaviour.
- **`can_fire_at_enemy_evaluator`** — may the creature shoot from where it is.
- **`idle_time_interval_passed_evaluator`** — has the idle dwell expired.
- **`lookout_time_interval_passed_evaluator`** — has the lookout dwell expired.
- **`loophole_planner_const_evaluator`** — a fixed answer, used to pin a world-state
  property.

## Notes

`in_cover_evaluator` and `cover_entered_evaluator` are two names for one test. They exist
separately because the planner's world-state properties are identified by *which evaluator
produced them*, so two distinct properties that happen to have the same truth condition
need two evaluator types. A rebuild whose world state is keyed by property name rather than
by evaluator identity needs only one.

The two dwell evaluators declare a mutable interval that is *rewritten during evaluation*.
That makes them stateful, which an evaluator normally is not — see
[`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md).
