# src/xrGame/stalker_movement_manager_space.h

> The bit-mask vocabulary that names a stalker's locomotion state, so that one
> integer selects an animation set and a speed.

**Needs** — _(none beyond the platform integer width)_
**Used by** — [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) · [`stalker_movement_manager_base.h`](stalker_movement_manager_base.h.md) · [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md) · [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md)
**Tier floor** — T3: a set of named constants; nothing in it is device- or format-facing.
The 32-bit width is load-bearing only because two flags live at the top of the word.

## Purpose

A stalker's locomotion is the product of four independent choices — how fast (standing,
walking, running), what posture (standing, crouching), what mental state (free, danger,
panic) and, for some combinations, whether the animation plays forwards or backwards. The
animation tables in the game data are keyed by the *combination*, not by the four
components. This file gives each component a bit and each shipped combination a name, so
that one integer is both a key and a decomposable state.

It is a vocabulary file: there is no behaviour here, and its value to a rebuild is the
list of which combinations actually exist in the shipped data.

## State

```text
ENUM VelocityMask   # a 32-bit word of independent flags
  # movement type — mutually exclusive in practice
  Standing        = bit 0
  Walk            = bit 1
  Run             = bit 2
  MovementTypeAll = Standing | Walk | Run        # mask for extracting the component

  # body state
  Stand           = bit 3
  Crouch          = bit 4
  BodyStateAll    = Stand | Crouch

  # mental state
  Danger          = bit 6                        # bit 5 is unused
  Free            = bit 7
  Panic           = bit 8
  MentalStateAll  = Danger | Free | Panic

  # animation direction, set on top of a combination
  PositiveVelocity = bit 31
  NegativeVelocity = bit 30
```

**Invariants** — exactly one bit from each of the three groups is set in a well-formed
mask; at most one of the two direction bits. Bit 5 is skipped, with no recoverable reason:
the mental-state group simply starts at bit 6.

## The named combinations

The enumeration then names every combination the shipped animation data actually contains.
That list is itself the specification — a rebuild must support these and need not support
the rest:

- standing still, in each of the three mental states, standing or crouching (six);
- walking free standing, walking in danger standing or crouching (three);
- running free standing, running in danger standing or crouching, running panicked
  standing (four).

Notice the asymmetries, which are authored facts rather than oversights: there is no
walking panic state, no crouched free movement other than standing still, and no crouched
panic movement other than standing still.

Each of the crouched-standing-still combinations and each of the walk and run combinations
appears again twice more, once with the positive-velocity bit and once with the
negative-velocity bit, giving the forward-playing and backward-playing variants. Standing
free and standing danger *while standing upright* have no directional variants, because
there is nothing to reverse.

**Notes**

The combination constants are formed by combining the component bits, so a rebuild is free
to compute them rather than list them, as long as the set of *legal* combinations is
enforced somewhere: the animation lookup fails on an unnamed combination, and that failure
is how authoring errors surface.
