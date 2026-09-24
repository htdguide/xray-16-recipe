# src/xrGame/script_object.cpp

> Composes the client-object lifecycle with the scripted-entity mixin's, and fixes the order in which the two halves run.

**Needs** — [`script_object.h`](script_object.h.md) · [`GameObject.h`](GameObject.h.md) · [`script_entity.h`](script_entity.h.md)
**Used by** — [`script_object.h`](script_object.h.md)
**Tier floor** — T2

## Purpose

Every method here is two calls. The file exists to answer one question per lifecycle stage:
**which half goes first**. Those orderings are the entire content of the file and they are
not uniform, which is the part worth writing down.

## State

`Stateless.` All state belongs to the two halves being composed.

## Lifecycle composition

**Contract** — each stage runs the base client object's half and the scripted-entity
mixin's half, in the order below.

```text
construct_parts      : client object, THEN mixin
reinit               : mixin,         THEN client object      # reversed
spawn_from(record)   : client object, AND THEN mixin — but only if the first succeeded
destroy_network_side : client object, THEN mixin
scheduled_update(dt) : client object, THEN mixin
frame_update         : client object, THEN mixin
```

**Invariants**

- **`reinit` runs in the opposite order to everything else.** The mixin's reset abandons
  the running action queue and releases the sound and particle handles it holds; those
  handles are registered with the client object's own tables, so they must be given back
  before the client object clears those tables. A rebuild that makes the order uniform
  leaks them.
- **Spawning is conjunctive and short-circuiting.** If the client object refuses the server
  record, the mixin is never initialized, and the entity as a whole reports failure so the
  caller discards it. A mixin initialized against a half-spawned object would hold a
  reference to something about to be thrown away.
- Nothing here is conditional on script presence. A scripted object with no bound script
  still runs the whole lifecycle; the mixin simply has an empty queue.

## `uses_navigation_positions`

**Contract** — answers *no*, unconditionally.

**Notes**

This is the file's one real decision. A scripted object does not occupy a navigation
vertex: it is not a creature, it does not path, and claiming a vertex would make it an
obstacle to creatures that do. The consequence is that a script moving one of these
through a corridor may drive it through geometry — which is the intended trade, since these
objects exist for set-pieces where the author is in control.

## `as_script_entity`

**Contract** — identifies this object as one carrying the scripted-entity mixin, so that a
caller holding only a client object can reach the action queue. The engine's general
"is this object also an X" facility; see
[`GameObject.h`](GameObject.h.md).
