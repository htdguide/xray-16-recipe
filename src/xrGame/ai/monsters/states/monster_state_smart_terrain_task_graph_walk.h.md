# src/xrGame/ai/monsters/states/monster_state_smart_terrain_task_graph_walk.h

> Declares the cross-level walk toward a smart-terrain job, implemented in
> [`monster_state_smart_terrain_task_graph_walk_inline.h`](monster_state_smart_terrain_task_graph_walk_inline.h.md).

**Needs** — [`monster_state_move.h`](monster_state_move.h.md) · [`monster_state_smart_terrain_task_graph_walk_inline.h`](monster_state_smart_terrain_task_graph_walk_inline.h.md)
**Used by** — [`monster_state_smart_terrain_task_graph_walk_inline.h`](monster_state_smart_terrain_task_graph_walk_inline.h.md) · [`monster_state_smart_terrain_task_inline.h`](monster_state_smart_terrain_task_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the first stage of serving a smart-terrain job: walking on the coarse game graph until the
creature occupies the same game-graph vertex as the job. It derives from the movement base in
[`monster_state_move.h`](monster_state_move.h.md) so that the path builder is prepared on entry.

Its private state is the task it is heading for, re-read from the server record each time the leaf
is offered or entered.

## `CStateMonsterSmartTerrainTaskGraphWalk`

- **enter** — read the assigned task
- **execute** — tour graph points toward the task's game-graph vertex, walking
- **is_startable** — a smart terrain is assigned and the creature is not already at its vertex
- **is_finished** — the creature occupies the task's game-graph vertex
