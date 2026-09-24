# src/xrGame/ai/monsters/states/monster_state_smart_terrain_task_graph_walk_inline.h

> Walk on the coarse cross-level graph until you are in the same graph cell as the job.

**Needs** — [`monster_state_smart_terrain_task_graph_walk.h`](monster_state_smart_terrain_task_graph_walk.h.md) · [`../../../alife_simulator.h`](../../../alife_simulator.h.md) · [`../../../../xrServerEntities/alife_monster_brain.h`](../../../../xrServerEntities/alife_monster_brain.h.md)
**Used by** — [`monster_state_smart_terrain_task_graph_walk.h`](monster_state_smart_terrain_task_graph_walk.h.md)
**Tier floor** — T3: reads the creature's own server record and moves on the coarse graph

## Purpose

The long-haul stage of serving an alife job. It exists because the two navigation graphs answer
different questions: the level mesh can path anywhere inside one level but knows nothing outside
it, while the game graph spans the whole game at the resolution of a few thousand places. A job
assigned in a distant part of the region is unreachable as a level-mesh destination and trivially
reachable as a game-graph tour.

The leaf therefore moves the creature *coarsely* until it shares a game-graph vertex with the job,
and then hands over to the in-level walk, which does the metre-accurate part.

## State

```text
RECORD GraphWalkState
  task : task   # read from the server record; only its game-graph vertex is used here
```

**Invariant** — the task is refreshed by both the start test and the entry step, so the leaf's
target is never older than the moment it was offered.

## `check_start_conditions`

**Contract** — startable when the creature's alife record still names an assigned smart terrain,
and the creature's current game-graph vertex is not already the job's.

```text
FUNCTION check_start_conditions() -> bool
  record = alife.entity_for(self.id)
  IF record.assigned_smart_terrain is none  RETURN false
  task = record.brain.assigned_task
  RETURN self.game_vertex != task.game_vertex
```

**Notes** — the second clause is what lets the composite above skip this stage entirely, which is
the common case: most jobs are in the same coarse cell the creature is already in.

The task is *assigned* to the field here, inside a predicate. As with the composite's own start
test, the predicate is not pure, and a rebuild that treats it as pure loses the target.

## `execute`

**Contract** — request the walking action and the idle voice, and tell the path component to tour
graph points in the direction of the task's game-graph vertex.

**Notes** — this is the same graph-point tour the peacetime wander uses
([`monster_state_rest_walk_graph_inline.h`](monster_state_rest_walk_graph_inline.h.md)), with one
difference: the wander gives it no destination and it drifts, while this gives it a target vertex
and it converges. One path-component mechanism, two behaviours, distinguished by one argument.

The tour is restated every update rather than issued once, so a job reassigned to a different
region is followed without any reset — although in practice the composite above forces a restart on
reassignment anyway.

The creature walks. There is no run variant and no acceleration profile: crossing a region toward a
job is never urgent, and a running animal crossing the whole map would read wrong.

## `check_completion`

**Contract** — finished when the creature's game-graph vertex equals the task's.

**Notes** — the test is on graph identity, not on distance, which is why this stage can end with
the creature still a long way from the job in world units. That is correct and is the division of
labour: this stage delivers the creature to the right coarse cell and the next one walks it to the
exact spot.
