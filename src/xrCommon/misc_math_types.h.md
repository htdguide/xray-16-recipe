# src/xrCommon/misc_math_types.h

> One record: an orientation as three angles, in the order the engine's cameras and creature heads use.

**Needs** — _(none)_
**Used by** — [`xr_object.h`](../xrEngine/xr_object.h.md)
**Tier floor** — T3: three numbers and a naming convention.

## Purpose

The engine represents "which way something is facing" three different ways depending on
what is facing: a direction vector, a rotation matrix, and — for anything an animation or a
control loop turns incrementally — three Euler angles. This file holds the third. It is a
separate file because it is needed by both the game layer and the engine layer and belongs
to neither.

## State

```text
RECORD SRotation
  yaw   : real    # radians, rotation about the vertical axis
  pitch : real    # radians, rotation about the lateral axis, positive is down
  roll  : real    # radians, rotation about the forward axis; usually zero

  # Default construction is all three zero, meaning the engine's reference
  # facing. There is no "unset" orientation - callers that need one carry a
  # separate flag.
```

**Invariants**

```text
# The components are NOT kept normalized.
#   Fields accumulate freely and go outside a single turn. Every consumer that
#   cares normalizes at the point of use with the angle helpers in math_funcs.h.
#   A rebuild that normalizes on write changes behaviour: the turn-toward loops
#   read the accumulated value to decide which way is shorter.

# The order is yaw, then pitch, then roll, applied in that order.
#   This is the order the animation system, the camera and the creature turn
#   controllers all assume. It is stated nowhere in the source and is inferred
#   from the conversions to and from matrices. A rebuild that picks a different
#   composition order gets creatures that look in the wrong direction only when
#   pitched, which is the hardest version of this bug to find.
```

**Notes** — the record is copied by value everywhere and has no behaviour of its own; the
constructors present in the source are C++ boilerplate and survive as "default is zero".
