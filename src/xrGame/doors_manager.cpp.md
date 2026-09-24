# src/xrGame/doors_manager.cpp

> The level's door registry: a spatial index of every door, and the query that hands one creature the doors it is about to have an opinion about.

**Needs** — [`doors_manager.h`](doors_manager.h.md) · [`doors_door.h`](doors_door.h.md) · [`doors_actor.h`](doors_actor.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a spatial query and delegation

## Purpose

Doors matter to the AI for one reason: a creature pathing through a building must open the
ones in its way and must not walk into the ones swinging open beside it. Answering that per
creature per frame against every door on a level would not scale, so this file owns a
spatial index and the radius query against it. Everything else here is one-line delegation
to the door record — the manager is deliberately thin, and its value is the index and the
radius.

## State

```text
RECORD DoorsManager
  doors         : quadtree<Door>     # spatial index over the whole level bounding box
  nearest_doors : list<Door>         # scratch, reused across queries
```

**Invariant** — the index is built once from the level's bounding box and never rebuilt. A
door that *moves* — and every door moves — keeps the position it was registered at; see the
registered-position field in [`doors_door.cpp`](doors_door.cpp.md). This is correct because
a door's hinge does not move, only its leaf.

**Invariant** — the registry must be empty when the level unloads. A door still registered at
teardown is a leaked physics object reference and is checked for.

**Notes** — the scratch list is a member rather than a local so that the per-frame query
allocates nothing. It makes the manager non-reentrant, which is fine because creature updates
are serialized.

## `register_door` · `unregister_door`

**Contract** — wraps a physics object in a door record, inserts it into the index, and
returns the handle; and the reverse, which removes from the index and destroys the record.
The manager owns the record, and unregistration clears the caller's handle so it cannot be
used afterwards.

**Notes** — the file carries a commented-out diagnostic that looked up one specific door by
authored name after every insertion and removal. It is the fossil of a hunt for an index
corruption bug and is worth noticing only because it says the index *was* once wrong.

## `actualize_doors_state`

**Contract** — the per-creature entry point, called from that creature's movement update.
Collects the doors within the reach radius and hands them to the creature's agent. Returns
whether the creature may proceed: false means it is waiting on a door.

```text
FUNCTION actualize_doors_state(agent, average_speed) -> bool
  radius = average_speed * g_door_open_time + g_door_length
  nearest = doors within `radius` of the agent's position
  IF nearest is empty AND the agent has nothing pending THEN RETURN true
  RETURN agent.update_doors(nearest, average_speed)
```

**Invariants** — the radius scales with the creature's speed, not with a fixed distance. A
sprinting creature must notice a door further out than a walking one, because the door takes
the same time to open either way. The added leaf length covers the door's own reach.

**Invariants** — the early exit requires *both* that no door is near and that the agent has
nothing pending. An agent still holding a door open must be updated even with no door in
range, or the door is never released. See the note on that test in
[`doors_actor.cpp`](doors_actor.cpp.md), which is where the condition is actually
implemented and where it is wrong.

## `on_door_is_open` · `on_door_is_closed`

**Contract** — the world's report that a door has finished moving, forwarded to the record's
state machine. These are the *inputs* to the machine; the engine never observes a door's
angle itself.

## `lock_door` · `unlock_door` · `is_door_locked`

**Contract** — the script-facing lock. Locking is a single flag on the record; the query
asks whether it is locked with respect to *either* state, which is the same as asking
whether it is locked at all and not already where you want it. Creatures treat a locked door
as an obstacle to wait at rather than one to route around.

## `is_door_blocked`

**Contract** — whether some creature is currently holding the door in a state, with respect
to either state — that is, whether anybody at all has claimed it. A blocked door is one
another creature is using, which a second creature must queue behind rather than fight over.

## `open_door` · `close_door`

**Contract** — declared and reachable only by the creature agent, which is a friend. They
are the manager's half of the agent/door relationship and exist to keep the agent from
reaching the record directly.
