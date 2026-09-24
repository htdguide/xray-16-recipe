# src/xrGame/stalker_combat_planner.h

> Declares the combat branch of a stalker's brain — the largest planner in the engine.

**Needs** — [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) · [`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md) · [`stalker_planner.cpp`](stalker_planner.cpp.md)
**Tier floor** — T2: a twenty-operator planner that is itself an operator, and is saved with the game.

## Purpose

Declares the surface implemented in
[`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md). It is the only sub-planner in
the stalker that is serialized, because a creature saved mid-firefight must resume its plan
rather than restart it.

## Exported units

- `POST_COMBAT_WAIT_INTERVAL` — three seconds. How long after the last enemy disappears the
  creature still counts as having one. Public because the root planner's enemy evaluator
  uses the same number, so the two agree by construction rather than by two copies.
- `setup(creature, parent_property_storage)` — rebuild the tables, seed the sequence
  propositions, subscribe to cover changes, and hand the movement layer the parent storage.
- `initialize()` / `execute()` / `update()` / `finalize()` — the branch lifecycle.
- `on_best_cover_changed(new, old)` — a subscription callback: the creature's chosen cover
  point moved.
- `save(packet)` / `load(reader)` — serialization, delegated to the planner base.

It holds three fields — the last enemy identity, a timestamp and a wounded flag — of which
only the wounded flag is read, by the enemy evaluator it is handed to.
