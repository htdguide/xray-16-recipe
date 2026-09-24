# src/xrServerEntities/alife_monster_brain.h

> Declares the offline brain every creature record carries: the thing that picks a smart terrain, asks it for a job, and walks the game graph towards it.

**Needs** — [`alife_space.h`](alife_space.h.md) · [`xrServer_Space.h`](xrServer_Space.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`alife_monster_brain_inline.h`](alife_monster_brain_inline.h.md)
**Used by** — [`monster_state_smart_terrain_task_graph_walk_inline.h`](../xrGame/ai/monsters/states/monster_state_smart_terrain_task_graph_walk_inline.h.md) · [`monster_state_smart_terrain_task_inline.h`](../xrGame/ai/monsters/states/monster_state_smart_terrain_task_inline.h.md) · [`alife_monster_abstract.cpp`](../xrGame/alife_monster_abstract.cpp.md) · [`alife_monster_base.cpp`](../xrGame/alife_monster_base.cpp.md) · [`alife_monster_brain_script.cpp`](../xrGame/alife_monster_brain_script.cpp.md) · [`alife_monster_detail_path_manager.cpp`](../xrGame/alife_monster_detail_path_manager.cpp.md) · [`alife_human_brain.h`](alife_human_brain.h.md) · [`alife_monster_brain.cpp`](alife_monster_brain.cpp.md) · [`alife_monster_brain_inline.h`](alife_monster_brain_inline.h.md) · [`xrServer_Objects_ALife_Monsters.cpp`](xrServer_Objects_ALife_Monsters.cpp.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_Objects_ALife_Monsters_script4.cpp`](xrServer_Objects_ALife_Monsters_script4.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in
[`alife_monster_brain.cpp`](alife_monster_brain.cpp.md).

## Exported units

- **the monster brain** — owned by a creature's *server* record, not by its live object.
  It runs while the creature is offline, and it keeps running (idly) while it is online.
- `update(forced)` — one offline decision cycle.
- `select_task(forced)` — choose a smart terrain, subject to a rate limit.
- `on_state_write` / `on_state_read` — the brain's contribution to the creature's record;
  empty at this level, overridden in [`alife_human_brain.h`](alife_human_brain.h.md).
- `on_register` / `on_unregister` / `on_location_change` / `on_switch_online` /
  `on_switch_offline` — the lifecycle hooks the simulation calls.
- `smart_terrain()` — resolve the creature's recorded smart-terrain identifier to the
  record, with a one-entry cache.
- `perform_attack()` / `action_type(...)` — what this creature does when the simulation
  decides it has met another; both answer "nothing" here and are overridden by the human
  brain.
- `can_choose_alife_tasks(value)` — a script-settable veto on the whole task-selection
  behaviour.
