# src/xrGame/ik_object_shift.h

> Declares the smoother that moves a character's whole body up or down so its feet can reach the ground.

**Needs** — [`ik_object_shift.cpp`](ik_object_shift.cpp.md) · [`pose_extrapolation.h`](pose_extrapolation.h.md)
**Used by** — [`IKLimbsController.cpp`](IKLimbsController.cpp.md) · [`IKLimbsController.h`](IKLimbsController.h.md) · [`ik_object_shift.cpp`](ik_object_shift.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`ik_object_shift.cpp`](ik_object_shift.cpp.md), which
holds the substance.

## State

```text
RECORD ObjectShift
  current      : real   # the shift value at `current_time`
  target       : real   # where it is heading
  target_time  : real   # when it should arrive
  current_time : real   # when the current trajectory was started
  speed        : real   # the trajectory's initial rate
  accel        : real   # ... its initial second derivative
  aaccel       : real   # ... and its constant third derivative (jerk)
  frozen       : bool   # when set, the shift is held at zero and targets are refused
```

Exported units:

- `reset` — return to zero, now, with no motion.
- `set_taget` — aim for a value at a time in the future.
- `shift` — the current value.
- `freeze` — suspend the mechanism entirely.
