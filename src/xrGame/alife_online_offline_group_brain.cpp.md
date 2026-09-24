# src/xrGame/alife_online_offline_group_brain.cpp

> The squad's offline decision loop: ask the smart terrain what this squad is supposed to be doing, and walk there.

**Needs** — [`alife_online_offline_group_brain.h`](alife_online_offline_group_brain.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) · [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: one delegation per tick

## Purpose

Every offline entity that moves has a brain; this is the squad's, and it is the simplest
one in the game. A squad has no personal motivations, no danger model and no planner: it
goes where its **smart terrain** job says to go. Everything a squad appears to decide is
decided elsewhere — by the smart terrain that issued the job, and by the scripts that
choose which smart terrain a squad belongs to.

The file is separate from the squad record because the *record* is what the save format
and the registries see, while the brain is the per-tick behaviour; a rebuild could merge
them, and the only cost is that the record's page would then mix serialization with
behaviour.

## State

```text
RECORD GroupBrain
  object           : ref OnlineOfflineGroup    # the squad this drives
  movement_manager : MovementManager           # owned, created with the brain
```

The movement manager exists for the brain's whole lifetime. Nothing else is stored:
the squad's position, its destination and its path all live one level down.

## `update`

**Contract** — one call per alife tick, from the squad's update, and only while the squad
is offline and non-empty. Fails hard when the squad has no current task.

```text
FUNCTION update()
  task = object.get_current_task()
  REQUIRE task exists  ELSE FAIL WITH "registered in a smart terrain but given no task"
  movement.path_type = GAME_PATH
  movement.detail.target(task)         # the task carries the destination triple
  movement.update()
```

**Invariants** — the movement mode is reasserted **every tick**, not set once. That is
deliberate: a script may have put the squad on a patrol path between ticks, and the brain
overrides it, because the squad's smart-terrain job is the authority on where the squad
goes. A rebuild that sets the mode once at formation would let a stale script assignment
win.

The destination is likewise re-pushed every tick rather than only when it changes. The
detail mover treats an unchanged destination as free (its cached path stays valid), so
this costs nothing and removes the need for any change detection.

**Notes** — the missing-task case is fatal rather than recoverable. The reasoning is that
a squad only has a brain running while it is registered with a smart terrain, and a smart
terrain that accepts a registration without issuing a job has already broken its own
contract; continuing would leave the squad walking to wherever it last went, forever,
which is far harder to diagnose than an immediate failure. A rebuild should keep it fatal.

## `on_switch_online` / `on_switch_offline`

**Contract** — forwarded to the movement manager, which forwards them to the detail mover.
These are the hooks that convert between the squad's coarse graph position and a real
position on a loaded level.

## Hooks the group brain declines

**Contract** — `on_state_write`, `on_state_read`, `on_register`, `on_unregister` and
`on_location_change` all do nothing.

**Notes** — the brain has no state of its own to serialize: the squad's destination is
recomputed from its smart-terrain job on the first tick after a load, and its path is a
cache. That is why a saved squad resumes correctly without the brain writing a byte, and
it is a design decision worth stating — *derive, do not persist* — rather than an
omission. The empty hooks exist because the surrounding machinery calls them on every
brain uniformly; a rebuild with optional hooks simply does not implement them.
