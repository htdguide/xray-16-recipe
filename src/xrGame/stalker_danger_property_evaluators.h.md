# src/xrGame/stalker_danger_property_evaluators.h

> Declares the questions a stalker's danger branch may ask about the threat it has selected.

**Needs** — [`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) · [`property_evaluator.h`](property_evaluator.h.md) · [`property_evaluator_const.h`](property_evaluator_const.h.md) · [`property_evaluator_member.h`](property_evaluator_member.h.md) · [`danger_object.h`](danger_object.h.md) · [`wrapper_abstract.h`](wrapper_abstract.h.md)
**Used by** — [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_danger_by_sound_planner.cpp`](stalker_danger_by_sound_planner.cpp.md) · [`stalker_danger_grenade_planner.cpp`](stalker_danger_grenade_planner.cpp.md) · [`stalker_danger_in_direction_planner.cpp`](stalker_danger_in_direction_planner.cpp.md) · [`stalker_danger_planner.cpp`](stalker_danger_planner.cpp.md) · [`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) · [`stalker_danger_unknown_planner.cpp`](stalker_danger_unknown_planner.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md)
**Tier floor** — T2: one small object per question per creature.

## Purpose

Declares the surface implemented in
[`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md).

It also names the three *shapes* every stalker evaluator takes, by binding the generic
evaluator bases to the stalker type: a plain evaluator, a constant one, and one that
answers about a squad member rather than about the creature itself. The binding exists
because each evaluator needs to reach the creature through both the engine handle and the
script-visible game object, and doing that in one place is what keeps every evaluator in
the stalker a three-line class.

## Exported units

- `Dangers` — is any danger selected at all.
- `DangerUnknown` — is the selected danger of a kind with no usable direction.
- `DangerInDirection` — is it of a kind that tells you where to look.
- `DangerWithGrenade` — is it a grenade.
- `DangerBySound` — is it a sound. Currently answers false unconditionally; see the
  implementation twin.
- `DangerUnknownCoverActual` — is the cover point the creature is heading to still the best
  one for this danger. The only stateful evaluator here; it remembers the position the
  current choice was made from.
- `DangerGrenadeExploded` — has the grenade that caused this danger already gone off.
- `GrenadeToExplode` — is there a grenade still in flight worth reacting to.
- `EnemyWounded` — is the current enemy a downed but not dead creature.
