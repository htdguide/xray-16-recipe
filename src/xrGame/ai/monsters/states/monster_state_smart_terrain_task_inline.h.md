# src/xrGame/ai/monsters/states/monster_state_smart_terrain_task_inline.h

> Serve the job the alife simulation assigned: walk across the game graph to the right region, then
> across the level to the exact cell, then sit there waiting to be taken in charge.

**Needs** — [`monster_state_smart_terrain_task.h`](monster_state_smart_terrain_task.h.md) · [`monster_state_smart_terrain_task_graph_walk.h`](monster_state_smart_terrain_task_graph_walk.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_data.h`](state_data.h.md) · [`../../../alife_simulator.h`](../../../alife_simulator.h.md) · [`../../../alife_object_registry.h`](../../../alife_object_registry.h.md) · [`../../../../xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../../../../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`../../../../xrServerEntities/alife_monster_brain.h`](../../../../xrServerEntities/alife_monster_brain.h.md)
**Used by** — [`monster_state_smart_terrain_task.h`](monster_state_smart_terrain_task.h.md)
**Tier floor** — T3: reaches across the server/client divide to read the entity's own alife record

## Purpose

The one behaviour in this directory that reaches *out* of the client object and into the
authoritative server record. Everything else here reads the creature's own memory components; this
one looks up the creature's alife entity by identity, asks its offline brain which smart terrain it
has been assigned to, and drives the live creature to that job's position.

That crossing is the whole design. Job assignment is an **offline** decision — made by the alife
simulation, on the coarse graph, for every creature in the world including the ones nobody is
looking at. This composite is what an *online* creature does to catch up with a decision that was
made about it while it was offline. It is why a player walking into a region finds animals already
converging on the places the world says they belong.

## State

```text
RECORD SmartTerrainTaskState
  current_task : optional<task>   # identity only; compared, never dereferenced for content
```

**Invariant** — `current_task` is the task in force when the composite entered or last restarted.
Every update compares it against the assignment now in effect; a difference means the simulation
reassigned the creature and the composite must restart from the beginning.

## `check_start_conditions`

**Contract** — startable only when the alife simulation is running; never for one particular
phantom entity class; and then only after asking the creature's offline brain to (re-)select a
task, when a smart terrain has been assigned and the creature has not yet been recorded as having
reached it.

```text
FUNCTION check_start_conditions() -> bool
  IF alife is not running          RETURN false
  record = alife.entity_for(self.id)
  IF record is a psy-dog phantom   RETURN false
  record.brain.select_task()                   # side effect: may assign one
  IF record.assigned_smart_terrain is none     RETURN false
  IF record.task_reached                       RETURN false
  RETURN true
```

**Notes** — two things here are easy to lose in a rebuild.

*The test has a side effect.* Asking "can this behaviour start" makes the offline brain choose a
job. Because the peacetime cascade calls this every update for every resting creature, job
selection for the whole online population is driven from here. A rebuild that makes the predicate
pure, or that caches its answer, stops creatures from ever being assigned work.

*The phantom exclusion is a class exclusion, not a condition.* One creature type is a conjured
duplicate with an alife record it does not really own; sending it to do a job would make the
illusion walk off. The check is by class, and a rebuild that expresses "is this entity real" some
other way must still exclude it.

Dedicated-server and no-alife configurations return false immediately, so the whole job system is
absent there.

## `check_completion`

**Contract** — finished when the assignment has been withdrawn, or when the creature is recorded as
having reached the job.

**Notes** — the arrival flag lives on the **server record**, not here, and is set by the smart
terrain when it takes the creature in charge. So the composite does not decide it has arrived; it
waits to be told. That is what the third leaf is for.

## `check_force_state` — reassignment

**Contract** — every update, before anything else: if the assignment is gone or the arrival flag is
now set, jump straight to the waiting leaf. Otherwise, if the task now in force is not the one
recorded, force the current leaf to a hard stop, clear the leaf bookkeeping so the sequence restarts
from the beginning, and adopt the new task.

```text
FUNCTION check_force_state()
  record = alife.entity_for(self.id)
  IF record.assigned_smart_terrain is none OR record.task_reached
    select(wait); RETURN

  task = record.brain.assigned_task
  IF task is none OR task != current_task
    IF a leaf is current  current_leaf.force_stop()
    clear current and previous leaf
    current_task = task
```

**Notes** — this is the composite's most important routine and it is what makes the behaviour
robust against a simulation that changes its mind. Smart terrains gain and lose capacity as other
creatures arrive and die, and a creature halfway across a level can have its job taken away and
another handed to it. Restarting the sequence — rather than retargeting the current leaf — is
correct, because a new job may be on a different level region and therefore needs the cross-level
walk again.

Clearing the *previous* leaf as well as the current one is what makes the reselection start from
its first branch rather than continuing the old sequence. That coupling between the two fields is
the mechanism, and it is invisible unless both are cleared.

## `reselect_state`

**Contract** — cross-level walk first if it will accept, otherwise in-level walk; then in-level
walk; then wait, absorbing.

```text
FUNCTION next_leaf(previous) -> leaf
  IF previous is none
    IF cross_level_walk.can_start()  RETURN cross_level_walk
    RETURN in_level_walk
  IF previous == cross_level_walk    RETURN in_level_walk
  RETURN wait
```

**Notes** — the two-stage walk mirrors the engine's two navigation graphs. The coarse game graph
gets the creature to the right *place in the world*; the level mesh gets it to the exact *cell*. A
job whose game-graph vertex the creature already occupies skips the first stage entirely, which is
the common case for a job inside the region the creature was already in.

## `initialize`

**Contract** — look up the creature's alife record and adopt whatever task its offline brain
currently holds.

**Notes** — the entry step asserts that a smart terrain has been assigned, relying on the start
condition having just established it. Entry and the start test are two separate traversals of the
same server record; a rebuild may fold them, but must keep the side-effecting task selection in the
predicate, since other callers rely on it.

## `setup_substates`

**Contract** — fill the parameters of the two leaves that take them. The cross-level walk
configures itself.

```text
in_level_walk:
  vertex           = task.level_vertex
  point            = navigation.position_of(that vertex)
  gait             = walk forward, accelerating, no braking, calm profile
  action time_out  = none               # walk however long it takes
  completion_dist  = 0                  # arrive exactly on the cell
  rebuild          = never              # keep the route planned on entry
  voice            = idle, delay = section key "idle_sound_delay"

wait:
  action           = rest
  time_out         = none               # absorbing
  voice            = idle, delay = section key "idle_sound_delay"
```

**Notes** — the walk uses the commit-to-one-route settings that recur throughout this directory
for destinations that do not move: no timeout, exact arrival, no re-planning. A job's cell is
fixed, so re-planning would only cost time.

*The waiting leaf is the handover point.* The creature has arrived and now rests at the job's cell
until the smart terrain sets the arrival flag on its server record, which the completion test then
sees. Nothing in this composite ever sets that flag, and nothing here times out waiting for it. A
creature whose smart terrain has become busy in the meantime therefore rests at the spot until the
force check notices the assignment is gone — which is precisely why the force check exists and why
it routes to this leaf rather than ending the composite.

`idle_sound_delay` is the only authored number in the composite.
