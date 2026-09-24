# src/xrGame/material_manager.cpp

> Footsteps: pairs the material a creature is made of with the material it is standing on, and plays a step sound at the right interval from the right place.

**Needs** — [`material_manager.h`](material_manager.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`material_manager.h`](material_manager.h.md)
**Tier floor** — T3: a timer and a table lookup per creature per frame

## Purpose

Every creature that walks makes a noise, and the noise is a function of *two* materials: what
it is walking with and what it is walking on. The material library holds one description per
ordered pair; this file is the per-creature machinery that keeps both indices current, times
the steps, and plays the sound at the creature's feet.

## State

```text
RECORD MaterialManager
  object            : GameObject
  movement          : MovementControl     # reports the ground material
  self_material     : int (16-bit)        # what this creature is made of; from configuration
  last_material     : int (16-bit)        # what it is standing on; WRITTEN BY the movement control
  time_to_step      : real                # seconds until the next footstep; negative fires
  step_id           : int                 # which of the four sound slots was used last
  step_sounds       : four sound handles
  running           : bool
```

**Invariant** — the ground material field is **not written here**. Its address is handed to
the physics movement control at reinitialization, and the physics writes into it directly as
the creature's feet touch surfaces. A rebuild should invert this into the movement control
reporting a value, but must preserve the timing: the value is current as of the last physics
step, not the last frame.

**Invariant** — both material indices must be valid whenever a step is produced, and the pair
they name must exist in the library. A missing pair is a hard failure naming the ground
material, because an unlisted pairing means the material data is incomplete and the
alternative — silence — hides it.

**Invariant** — four sound slots, indexed by the *same* index used to choose the sound from
the material pair's list. This couples two unrelated things and is the file's one real bug —
see below.

## `Load`

**Contract** — reads the creature's own material name from its configuration section and
resolves it to an index. A section with no material is a hard failure naming the section: a
creature with no material would pair with nothing.

## `reinit`

**Contract** — per-spawn reset. Sets the ground material to `default` — so a creature that
has not yet touched anything still produces a sound — clears the step counter and the running
flag, and, for a living creature, wires the physics movement control: gives it the address to
report the ground material into, and tells it this creature's own material.

**Invariant** — the movement control must be told the creature's own material as well as
given the reporting address. The physics uses it for collision response — how the creature
sounds and scuffs when it is *hit*, not when it walks — which is a separate consumer of the
same value.

**Notes** — a non-living object carrying this manager skips the wiring entirely and its ground
material stays `default` forever. That is the intended behaviour for objects that are not
creatures.

## `update`

**Contract** — called every frame while the creature moves, with the frame delta, a volume
scale, the interval between steps, and whether the creature is standing still. Plays a
footstep when the timer expires and keeps every sounding footstep positioned at the creature's
feet.

```text
FUNCTION update(delta, volume, step_interval, standing)
  pair = material pair (self_material, last_material)    # must exist
  position = object position, raised by the foot radius when the creature has a body

  IF standing THEN
    time_to_step = 0                                     # armed to fire on the next step
  ELSE
    IF time_to_step < 0 THEN
      sounds = pair.step_sounds
      IF running AND pair.breaking_sounds is non-empty THEN sounds = pair.breaking_sounds
      IF sounds is non-empty THEN
        slot = a uniform random index into sounds
        time_to_step = step_interval
        step_sounds[slot] = sounds[slot]
        play it at position, attached to the object
    time_to_step = time_to_step - delta

  FOR EACH sounding slot
    move it to position ; set its volume to volume
```

**Invariants** — the step interval is supplied by the caller, not derived here. It comes from
the creature's animation, so footsteps land on the animation's contact frames rather than on a
fixed clock. Standing resets the timer to zero rather than to the interval, so the first step
after stopping fires immediately — which is what makes starting to walk sound prompt.

The four sound slots exist so that a footstep can still be ringing when the next one starts;
the loop that re-positions and re-scales every sounding slot is what keeps a step sound
attached to a creature that is still moving.

**Notes**

- Indexing the slot array with the index drawn from the *material's* sound list is wrong. The
  list can be longer than four, in which case the index is out of range and memory outside the
  array is written; and two consecutive steps drawing the same index cut each other off
  regardless of how many slots are free. A rebuild should draw the sound independently of the
  slot and allocate the slot round-robin among free ones. The shipped material data keeps its
  lists at four or fewer, which is why this has never been fixed.
- The running flag selects *breaking* sounds over step sounds when the pair declares any.
  Those are the heavier, more percussive variants — twigs, glass, gravel giving way — and a
  pair without them falls back to the walking set, so running on a surface with no special
  sounds simply sounds like walking faster.
- The sound is raised from the object's origin by the creature's foot radius, so it comes from
  the ground rather than from the model's pivot. An object with no physics body is left at its
  origin.

## `set_run_mode`

**Contract** — switches between the walking and running sound sets. Takes effect on the next
step, not immediately, because the running set is chosen at the moment a step fires.

## `reload`

**Contract** — present and does nothing. The material assignment is fixed at load and there is
no per-instance override for it, unlike the terrain preference in
[`location_manager.cpp`](location_manager.cpp.md). The empty override exists only to complete
the shape shared by the other per-creature managers. A rebuild omits it.
