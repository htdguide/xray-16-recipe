# src/xrGame/ActivatingCharCollisionDelay.cpp

> Retries creating a creature's walking collision capsule until it can be placed somewhere it does not overlap the world.

**Needs** — [`ActivatingCharCollisionDelay.h`](ActivatingCharCollisionDelay.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ActivatingCharCollisionDelay.h`](ActivatingCharCollisionDelay.h.md)
**Tier floor** — T2: a timer and a position correction; no layout concern

## Purpose

A living creature moves through the world as an upright capsule, not as a ragdoll. That
capsule cannot be created while the creature is standing inside geometry — the physics
solver would eject it violently or trap it. This object exists for the window in which a
creature is *supposed* to be walking but its capsule could not yet be created: it is
constructed at the moment of failure and keeps trying.

It is a small, deliberately stupid retry loop, and the reason it is its own file is that
the two obvious places for it — the physics support object and the movement control —
would both have to grow a timer that only matters during this one transient.

## State

```text
RECORD ActivatingCharacterDelay
  support       : reference to the creature's physics support   # non-owning, outlives this
  activate_time : int       # global-clock timestamp of the next attempt
  RETRY_PERIOD  : int = 3000 ms   # constant
```

**Invariants** — construction requires that the creature has a movement control and that
its capsule does *not* currently exist; the object is meaningless otherwise. The object is
non-copyable because it holds a reference to a live subsystem.

**Notes** — the three-second retry period is long enough that a failed attempt costs
nothing and short enough that a creature freed by a moving obstacle starts walking within
a step or two of the obstacle clearing. There is no derivation for the exact value.

## `active`

**Contract** — true while the capsule still does not exist. Becomes false permanently once
the capsule is created; the owner drops this object at that point.

## `update`

**Contract** — called on the creature's normal update path. Does nothing until the retry
timestamp is reached, then attempts one placement correction and, if the correction found
free space, creates the capsule. The timestamp is re-armed whether or not the attempt
succeeded, so a failure costs exactly one attempt per period.

```text
FUNCTION update()
  IF NOT active() THEN RETURN
  IF now < activate_time THEN RETURN

  IF correct_position() THEN
    support.create_character()          # the capsule now exists; active() goes false

  activate_time = now + RETRY_PERIOD
```

**Notes** — the clock read is the *global* clock, which pauses with the game. A creature
waiting to stand up does not accumulate retries while the game is paused, which is what a
player would expect.

## position correction (private)

**Contract** — asks the physics support to nudge the object out of any geometry it
overlaps, and reports whether it succeeded. On failure the position is restored exactly to
what it was before the attempt, because a partial correction leaves the creature
mid-geometry and visibly displaced for the next three seconds.

```text
FUNCTION correct_position() -> bool
  saved = object.position
  ok = support.collision_correct_object_position()
  IF NOT ok THEN object.position = saved      # never leave a half-applied nudge
  RETURN ok
```

**Notes** — the preconditions asserted here are the whole contract of the class in one
place: the object is a living entity, it is *alive*, and it has no ragdoll shell. A
ragdolled or dead creature must never grow a walking capsule, and this is where that is
enforced.
