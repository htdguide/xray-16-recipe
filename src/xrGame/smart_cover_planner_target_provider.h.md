# src/xrGame/smart_cover_planner_target_provider.h

> Declares the actions whose whole job is to hand the animation planner its next goal, plus the two that also decide when a creature has been firing for too long.

**Needs** — [`action_base.h`](action_base.h.md) · [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md)
**Used by** — [`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md) · [`smart_cover_planner_target_provider.cpp`](smart_cover_planner_target_provider.cpp.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md)
**Tier floor** — T2: planner actions that set another planner's goal

## Purpose

Declares the surface implemented in
[`smart_cover_planner_target_provider.cpp`](smart_cover_planner_target_provider.cpp.md).

The smart-cover system has two planners stacked: the target selector decides *what the
creature should be doing* and the animation planner decides *how to get there*. These
actions are the join. Each is an operator of the upper planner whose effect is to set the
lower planner's goal.

## `target_provider`

**Contract** — the base. Constructed with a world property; on initialization it makes
that property the animation planner's goal, marks it satisfied in its own planner's world
state, and decays the loophole preference score. Everything else delegates.

## `target_idle`

**Contract** — a provider that additionally clears the "firing too long" flag once its
inertia has run out. This is how a creature that has dropped back to idle becomes willing
to fire again.

## `target_fire`

**Contract** — a provider that sets its own inertia from the number of enemies, and raises
the "firing too long" flag when that inertia expires — unless the creature is nearly out of
ammunition, in which case it keeps firing.

## `target_fire_no_lookout`

**Contract** — a provider that clears the "looked out" flag before running, so blind fire
does not count as having peeked.

## Notes

`target_fire_no_lookout` re-declares the same two fields the base already holds and never
uses its own copies. A rebuild should drop them.
