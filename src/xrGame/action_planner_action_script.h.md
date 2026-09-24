# src/xrGame/action_planner_action_script.h

> Declares the bridge that lets an engine-written composite action live in a script-facing planner. Behaviour is in [`action_planner_action_script_inline.h`](action_planner_action_script_inline.h.md).

**Needs** — [`action_planner_action.h`](action_planner_action.h.md) · [`action_planner_action_script_inline.h`](action_planner_action_script_inline.h.md)
**Used by** — [`action_planner_action_script_inline.h`](action_planner_action_script_inline.h.md) · [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md) · [`stalker_alife_planner.h`](stalker_alife_planner.h.md) · [`stalker_anomaly_planner.h`](stalker_anomaly_planner.h.md) · [`stalker_combat_planner.h`](stalker_combat_planner.h.md) · [`stalker_danger_by_sound_planner.h`](stalker_danger_by_sound_planner.h.md) · [`stalker_danger_grenade_planner.h`](stalker_danger_grenade_planner.h.md) · [`stalker_danger_in_direction_planner.h`](stalker_danger_in_direction_planner.h.md) · [`stalker_danger_planner.h`](stalker_danger_planner.h.md) · [`stalker_danger_unknown_planner.h`](stalker_danger_unknown_planner.h.md) · [`stalker_death_planner.h`](stalker_death_planner.h.md) · [`stalker_get_distance_planner.h`](stalker_get_distance_planner.h.md) · [`stalker_kill_wounded_planner.h`](stalker_kill_wounded_planner.h.md) · [`stalker_low_cover_planner.h`](stalker_low_cover_planner.h.md) · _and 2 more_
**Tier floor** — T3: a declaration

## Purpose

The same bridge [`action_script_base.h`](action_script_base.h.md) provides for a leaf
action, provided for a composite one: the planner above it speaks in game object facades,
the engine code inside it wants the concrete creature, and this type holds both.

Substance is in
[`action_planner_action_script_inline.h`](action_planner_action_script_inline.h.md).

Exported units:

- Two constructors — with and without an initial precondition and effect set — taking the
  concrete object and deriving the facade from it.
- Two `setup` overloads, facade-taking and object-taking.
- `object` — the concrete creature.
