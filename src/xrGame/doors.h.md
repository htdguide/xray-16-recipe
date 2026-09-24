# src/xrGame/doors.h

> The vocabulary of the door subsystem: the two states a door can be in, and the two constants that size every decision about one.

**Needs** — _(none)_
**Used by** — [`GameObject.cpp`](GameObject.cpp.md) · [`doors_actor.h`](doors_actor.h.md) · [`doors_door.cpp`](doors_door.cpp.md) · [`doors_door.h`](doors_door.h.md) · [`doors_manager.h`](doors_manager.h.md)
**Tier floor** — T3: an enumeration and two constants

## Purpose

The shared header of the `doors` subsystem, kept separate so that the manager, the per-
creature agent and the door itself can each name the others' types without including each
other. That is a real dependency-breaking split and worth keeping.

## State

```text
ENUM DoorState
  open
  closed

g_door_length    : real = 1.1    # metres: the length of a door leaf, and the margin
                                 #   added to every proximity radius
g_door_open_time : real = 1.4    # seconds: how long a door takes to finish moving
```

**Invariant** — a door is *binary*. There is no half-open state in the model; a door that is
physically mid-swing is recorded as being in its old state until the world reports it has
arrived. Everything downstream — the blocking tests, the initiator bookkeeping — depends on
that, and a rebuild that models a continuous angle has to answer questions this design never
asks.

**Invariant** — these two constants are the subsystem's entire tuning, they are compiled in
rather than configured, and they combine in exactly one expression, which appears three
times:

```text
danger_distance = creature_speed * g_door_open_time          # how far I travel while it moves
detection_radius = danger_distance + g_door_length           # ... plus the leaf's own reach
```

The meaning is: *start dealing with a door when you are close enough that it would still be
moving when you arrive*. A rebuild that changes either constant changes how early creatures
reach for door handles, which is visible.

**Notes** — the leaf length is also used as the scale factor when a door's swing vectors are
built (see [`doors_door.cpp`](doors_door.cpp.md)), so the same 1.1 is simultaneously a
geometric length and a safety margin. That conflation is probably accidental but the two
uses are consistent.

## `doors_type`

**Contract** — the subsystem's list-of-doors alias. Doors are referred to by handle
throughout; the manager owns them and everything else borrows.
