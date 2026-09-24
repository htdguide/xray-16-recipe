# src/xrGame/script_particle_action.cpp

> Creates the live particle effect the channel owns — and does not destroy it.

**Needs** — [`script_particle_action.h`](script_particle_action.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

Two members that need the particle system's full definition: the destructor, and the setter
that instantiates an effect by name.

## `set_particle`

**Contract** — names the effect, tags the goal *attached*, creates a live effect instance
from the named authored effect, records whether that instance is self-removing, and clears
both the started and the completion flags.

```text
FUNCTION set_particle(name, auto_remove)
  effect_name = name
  goal_type   = attached
  auto_remove = auto_remove
  effect      = create particle effect from effect_name, self-removing IF auto_remove
  started     = false
  completed   = false
```

**Invariants** — the instance is created *now*, not when the action runs. A channel is
therefore not free to build, and a script that constructs one per frame allocates one
effect per frame.

## Destruction

**Contract** — the destructor releases nothing.

**Notes**

This is the file's one genuinely load-bearing fact, and it is an unresolved one in the
original: the channel creates an effect instance and never deletes it, with the deletion
commented out rather than removed. Two readings are consistent with the code, and the
source does not say which is intended:

- effects created with the self-removing flag delete themselves when they finish, and the
  non-self-removing ones are handed to the entity that plays the action, which then owns
  them; or
- the deletion was removed to stop a crash — freeing an effect the particle system was
  still stepping — and non-self-removing effects simply leak for the life of the level.

A rebuild must decide the ownership explicitly: either the channel owns the effect and
releases it on destruction after telling the particle system to drop it, or the channel
holds a borrowed handle and the effect's lifetime belongs to the world. What it must not do
is copy the original's silence here.
