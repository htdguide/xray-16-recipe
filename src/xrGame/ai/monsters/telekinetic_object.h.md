# src/xrGame/ai/monsters/telekinetic_object.h

> Declares one levitated object: its phase, its timers, its target height, and the sounds that follow it up and out.

**Needs** — [`telekinetic_object.cpp`](telekinetic_object.cpp.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`TeleWhirlwind.cpp`](../../TeleWhirlwind.cpp.md) · [`TeleWhirlwind.h`](../../TeleWhirlwind.h.md) · [`telekinesis.cpp`](telekinesis.cpp.md) · [`telekinesis.h`](telekinesis.h.md) · [`telekinetic_object.cpp`](telekinetic_object.cpp.md)
**Tier floor** — T2: a phase enumeration, four timestamps and two sound handles

## Purpose

Declares the surface implemented in
[`telekinetic_object.cpp`](telekinetic_object.cpp.md), and fixes the four-phase vocabulary
the whole telekinesis feature is expressed in.

```text
ENUM TelekineticPhase = { released, rising, hovering, thrown }
```

The phases are not symmetric. *Rising* and *hovering* are held states with a force applied
every physics step; *thrown* is ballistic, with the physics library in sole control and only
a timer running; *released* means the record is finished and its owner should drop it.

## State

```text
RECORD TelekineticObject
  phase         : TelekineticPhase
  object        : PhysicsObject           # the thing being lifted
  owner         : Telekinesis             # the controller that holds this record
  target_height : real (world units)      # absolute Y, computed once at activation
  strength      : real                    # per-object lift strength
  rotate        : bool                    # tumble the object while it rises

  time_rise_started  : int (milliseconds)
  time_keep_started  : int
  time_keep_updated  : int                # written, never read (see the implementation twin)
  time_to_keep       : int                # how long hovering lasts
  time_fire_started  : int

  sound_hold  : SoundHandle               # looped while held, follows the object
  sound_throw : SoundHandle               # one shot on a timed throw
```

**Invariants** — `target_height` is an absolute world height fixed at activation from the
object's position plus the requested lift. It does not track the object, so an object
activated on a table and then pushed off it will hover at table height plus the lift, not
at floor height plus the lift. Every phase transition stamps its own start time, so the
timestamps are only meaningful for the phase that is current.

## Exported units

- **initialise** — adopts an object, stamps the rising phase, computes the target height,
  switches the object's gravity off. Refuses an object with no physics body.
- **attach sounds** — clones a hold loop and a throw one-shot from the owner's prototypes.
- **per-step forces** — the rising force and the hovering force.
- **per-tick phase advance** — one entry point that dispatches on the phase.
- **phase transitions** — begin hovering, release, throw with a power factor, throw with a
  flight time, and the raw phase switch that stamps timestamps.
- **predicates** — reached the target height, rising timed out, hovering elapsed, flight
  elapsed, is released.
- **incidental helpers** — wake the physics body, tumble the object, test whether an object
  can be levitated at all, and compare a record against an object.
