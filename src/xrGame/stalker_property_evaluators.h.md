# src/xrGame/stalker_property_evaluators.h

> Declares the stalker's evaluator set and the three shapes an evaluator can take.

**Needs** — [`stalker_property_evaluators.cpp`](stalker_property_evaluators.cpp.md) · [`wrapper_abstract.h`](wrapper_abstract.h.md) · [`property_evaluator_const.h`](property_evaluator_const.h.md) · [`property_evaluator_member.h`](property_evaluator_member.h.md) · [`danger_object.h`](danger_object.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_property_evaluators_inline.h`](stalker_property_evaluators_inline.h.md)
**Used by** — [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md) · [`stalker_alife_planner.cpp`](stalker_alife_planner.cpp.md) · [`stalker_anomaly_planner.cpp`](stalker_anomaly_planner.cpp.md) · [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_death_planner.cpp`](stalker_death_planner.cpp.md) · [`stalker_get_distance_planner.cpp`](stalker_get_distance_planner.cpp.md) · [`stalker_kill_wounded_planner.cpp`](stalker_kill_wounded_planner.cpp.md) · [`stalker_low_cover_planner.cpp`](stalker_low_cover_planner.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md) · [`stalker_property_evaluators.cpp`](stalker_property_evaluators.cpp.md) · [`stalker_property_evaluators_inline.h`](stalker_property_evaluators_inline.h.md) · [`stalker_search_planner.cpp`](stalker_search_planner.cpp.md)
**Tier floor** — T2: a family of predicate types bound to one creature type.

## Purpose

Declares the surface implemented in
[`stalker_property_evaluators.cpp`](stalker_property_evaluators.cpp.md): twenty concrete
evaluators, plus the three aliases that say what kind of evaluator a stalker's questions
are built from.

## The three shapes

Before the list, the three bases. Each is the generic evaluator of chapter 14 specialized to
*a stalker*, so that a concrete evaluator is handed the creature rather than a script
handle:

- **measured** — the general case: an evaluator that inspects the creature and computes an
  answer. Every entry below except the two named exceptions is one of these.
- **constant** — an evaluator that ignores the world and returns a fixed answer. Used where
  a property must exist for an operator to refer to, but is meaningful only inside a deeper
  sub-planner, or where it is the never-satisfiable goal.
- **member** — an evaluator that reads a property out of *another* planner's world state
  rather than measuring anything. This is how a sub-planner inherits a fact its parent has
  already established, and it is the only channel between levels of the planner tree that
  does not go through the creature.

Every evaluator is constructed with the creature it is bound to and a human-readable name
that the planner's failure dump prints. Construction tolerates a missing creature — the
planner builds its tables before the creature is fully alive — so a rebuild that requires
the binding at construction must move table construction later.

## Exported units

- `ALife` — is the off-screen simulation running.
- `Alive` — is the creature alive.
- `Items` — has an item to pick up been chosen.
- `Enemies` — is there an enemy, or was there one within a supplied grace period; carries
  that period and an optional flag that cancels it.
- `SeeEnemy` — is the enemy visible now.
- `EnemySeeMe` — does the enemy see me now.
- `ItemToKill` / `ItemCanKill` — do I have a weapon, and is it usable.
- `FoundItemToKill` / `FoundAmmo` — do I remember a weapon, or ammunition, to go and fetch.
- `ReadyToKill` — can I fire now; carries an ammunition floor.
- `ReadyToKillSmartCover` — the same, relaxed for a cover with no firing loophole.
- `ReadyToDetour` — may I circle the enemy.
- `Anomaly` / `InsideAnomaly` — is there an unaccounted anomaly near me, or am I in one.
- `Panic` — should I break off and run.
- `SmartTerrainTask` — has a smart terrain given me a job (and, as a side effect, ask for
  one).
- `EnemyReached` — am I the squad's designated finisher and in reach.
- `PlayerOnThePath` — is the player blocking my way.
- `EnemyCriticallyWounded` — is my enemy down.
- `ShouldThrowGrenade` — should I throw now (and, as a side effect, record the target).
- `LowCover` — am I in cover too low to fire from. Disabled; always false.
