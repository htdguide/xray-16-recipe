# src/xrGame/pose_extrapolation.h

> Declares the two-sample pose history and the transform algebra it extrapolates with. Implemented in [`pose_extrapolation.cpp`](pose_extrapolation.cpp.md).

**Needs** — [`pose_extrapolation.cpp`](pose_extrapolation.cpp.md) · [`xrPhysics/CycleConstStorage.h`](../xrPhysics/CycleConstStorage.h.md)
**Used by** — [`IKLimbsController.cpp`](IKLimbsController.cpp.md) · [`IKLimbsController.h`](IKLimbsController.h.md) · [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`ik_object_shift.cpp`](ik_object_shift.cpp.md) · [`ik_object_shift.h`](ik_object_shift.h.md) · [`pose_extrapolation.cpp`](pose_extrapolation.cpp.md)
**Tier floor** — T2: a declaration plus a fixed-size ring

## Purpose

Something's transform is known at a coarse rate — sampled every fifty milliseconds — and is
needed at the frame rate. This is the machinery for guessing the intervening and following
values from the last two samples.

Three types, in layers:

- a **pose** — a position and an orientation, with the four operations extrapolation needs
  (scale, compose, invert, identity);
- a **point** — one pose stamped with the time it was taken;
- **points** — a ring of exactly two of those, plus the sampling policy and the
  extrapolation.

The ring size is fixed at **two**, which is what makes the extrapolation linear and what
makes the whole thing affordable. A commented-out alternative held three. Nothing else
about the shape is a decision.

Exported units:

- `pose` — `set` from a transform, `get` back to one, `mul` by a scalar, `add` another,
  `invert`, `identity`.
- `point` — `set` a pose with a time; read the pose and the time.
- `points` — `init` from a transform (fill both samples), `update` from a transform
  (subject to the sampling interval), `extrapolate` to a time.
