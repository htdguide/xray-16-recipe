# src/xrGame/moving_object.cpp

> One creature's presence in the world's obstacle-avoidance index: registered on construction, unregistered on destruction, reindexed whenever it moves.

**Needs** — [`moving_object.h`](moving_object.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`ai_space.h`](ai_space.h.md)
**Used by** — reached through its declarations in [`moving_object.h`](moving_object.h.md); callers name that, not this file.
**Tier floor** — T3: registry bookkeeping and forwarding

## Purpose

The creature-side half of the obstacle-avoidance system; the world-side half is
[`moving_objects.cpp`](moving_objects.cpp.md). This file is almost entirely lifecycle: it
exists so that the avoidance index's membership is maintained automatically by the
existence of the record, rather than by callers remembering to add and remove.

## State

```text
RECORD MovingObject
  creature         : reference to a living entity      # never absent
  ignored          : optional<reference to an object>  # one exemption from obstacle tests
  indexed_position : vector                            # may lag the creature's real position
  static_query     : ObstaclesQuery                    # level furniture in the way
  dynamic_query    : ObstaclesQuery                    # other creatures in the way
  action           : ActionType
  action_position  : vector
  action_frame     : int      # frame on which the action was last set
  action_time      : int      # wall-clock milliseconds when the action last *changed*
```

**Invariants** — the indexed position is the key the spatial index sorted this record
under. It is *not* kept equal to the creature's position; it is updated only through the
reindexing path, because changing it without removing and reinserting the record would
corrupt the index. Any rebuild that makes the position a live read of the creature has
broken the index.

The ignored reference is left uninitialized by the constructor. Reading it before the
first `ignore` call is a defect the original tolerates because the comparison it feeds
merely produces a wrong answer rather than a crash; a rebuild should initialize it to
absent.

## Construction and destruction

**Contract** — constructing the record fixes the creature, sets the action to *move* with
an unreachable sentinel position and zero frame and time, samples the creature's position,
and inserts the record into the world's avoidance index. Destroying it removes the record
from that index. Neither blocks.

```text
FUNCTION construct(creature)
  this.creature := creature                # required; absent is a programming error
  action := move
  action_position := (infinity, infinity, infinity)   # "no position recorded"
  action_frame := 0
  action_time := 0
  update_position()
  world avoidance index.register(this)

FUNCTION destroy()
  world avoidance index.unregister(this)
```

**Invariants** — register and unregister must pair exactly once each. The index asserts
both directions in checked builds — registering twice and unregistering an unknown record
are each reported by creature name — which is the local form of the engine-wide invariant
that no entity is registered twice.

The sentinel action position is the largest representable coordinate in all three axes, so
that any distance test against it fails. It means "the action has no position", encoded
without a separate flag.

## `on_object_move`

**Contract** — tells the world index this creature has moved. The index removes the record,
resamples the position and reinserts it. Called by the creature whenever its position
changes enough to matter, not every frame.

**Notes** — remove-resample-insert is the whole update, and the source marks it as an
optimization target: a spatial index that could move a record in place would avoid the
tree surgery. For a rebuild this is a free choice; nothing depends on the record being
absent between the two halves, because the call is not concurrent with a query.

## `update_position`

**Contract** — copies the creature's position into the record. Never call it outside the
reindexing path (see the state invariant).

## `predict_position` / `target_position`

**Contract** — pure forwards to the creature's own movement manager: where it will be after
*t* seconds of following its current path, and the final point of that path. The avoidance
system does all its reasoning on these two, never on raw velocity, which is why it can
resolve a collision a second before it happens.

## `ignore` / `ignored`

**Contract** — `ignore` records a single object to be exempted from this creature's obstacle
tests, replacing any previous exemption. `ignored` answers true for the creature itself and
for the exempted object, false otherwise.

**Notes** — self-exemption is unconditional and does not depend on `ignore` ever being
called. One slot is enough because the only use is "I am walking to this thing, so it must
not count as an obstacle" — a creature approaching a door, a companion following its
leader.
