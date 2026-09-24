# src/xrGame/ai/monsters/control_movement.h

> Declares the movement resource and its payload, implemented in [`control_movement.cpp`](control_movement.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`control_animation_base_accel.cpp`](control_animation_base_accel.cpp.md) · [`control_manager.cpp`](control_manager.cpp.md) · [`control_manager.h`](control_manager.h.md) · [`control_movement.cpp`](control_movement.cpp.md) · [`control_movement_base.cpp`](control_movement_base.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlMovement`, the pure element on the movement channel. Substance is in
[`control_movement.cpp`](control_movement.cpp.md).

## State

`SControlMovementData` — the channel payload: a target linear speed and the acceleration
to approach it with, where an unbounded acceleration means "reach it this frame".

Exported units:

- `reinit`, `update_frame` — reset; ease the speed and publish it.
- `velocity_current`, `velocity_target` — the eased speed and the commanded one.
- `real_velocity` — the speed the physics body is actually achieving, clamped. This is the
  number the animation base matches clips against.
