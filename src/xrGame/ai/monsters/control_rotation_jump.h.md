# src/xrGame/ai/monsters/control_rotation_jump.h

> Declares the rotation-jump ability and its payload, implemented in [`control_rotation_jump.cpp`](control_rotation_jump.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`bloodsucker.cpp`](bloodsucker/bloodsucker.cpp.md) · [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_manager_custom.h`](control_manager_custom.h.md) · [`control_rotation_jump.cpp`](control_rotation_jump.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlRotationJump`. Substance is in
[`control_rotation_jump.cpp`](control_rotation_jump.cpp.md).

## State

`SControlRotationJumpData` — the channel payload: a stop clip and a run clip for each side,
the angle the first stage turns through, and two flags (turn on the spot; use only the
first stage).

The four tuning constants — a cooldown window of three to five seconds, a facing check of
150 degrees and a speed tolerance of two units per second — are declared here as class
constants rather than read from configuration. They are the only numbers in this ability
that are not authored per creature.

Exported units:

- `reinit`, `activate`, `on_release`, `on_event`, `check_start_conditions` — the ability
  lifecycle.
- `build_line_first`, `build_line_second`, `stop_at_once` — private; the two deceleration
  and acceleration lines and the turn-in-place variant.
