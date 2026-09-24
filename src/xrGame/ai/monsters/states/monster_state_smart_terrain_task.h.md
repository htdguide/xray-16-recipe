# src/xrGame/ai/monsters/states/monster_state_smart_terrain_task.h

> Declares the go-do-the-job-a-smart-terrain-assigned composite, implemented in
> [`monster_state_smart_terrain_task_inline.h`](monster_state_smart_terrain_task_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_smart_terrain_task_inline.h`](monster_state_smart_terrain_task_inline.h.md) · [`../../../alife_smart_terrain_task.h`](../../../alife_smart_terrain_task.h.md)
**Used by** — [`group_state_rest_inline.h`](../group_states/group_state_rest_inline.h.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`monster_state_smart_terrain_task_inline.h`](monster_state_smart_terrain_task_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the highest-priority peacetime behaviour: the bridge between a creature's local brain and the
alife simulation's job system. It is what makes an authored place — a nest, a camp, a feeding
ground — actually pull creatures to it.

Its private state is the task currently being served, held as an identity to detect reassignment.

## `CStateMonsterSmartTerrainTask`

- **construct** — register three leaves: cross-level walk, in-level walk, and wait
- **enter** — record the assigned task
- **is_startable** — the simulation has assigned a smart terrain and the creature has not arrived
- **is_finished** — the assignment was withdrawn, or the creature has arrived
- **check_force_state** — abandon and restart when the assignment changes underneath
- **reselect** / **setup** — the three-step sequence and its parameters
