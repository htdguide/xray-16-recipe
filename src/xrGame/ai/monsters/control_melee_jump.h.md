# src/xrGame/ai/monsters/control_melee_jump.h

> Declares the melee-jump ability and its payload, implemented in [`control_melee_jump.cpp`](control_melee_jump.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_manager_custom.h`](control_manager_custom.h.md) · [`control_melee_jump.cpp`](control_melee_jump.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlMeleeJump`. Substance is in
[`control_melee_jump.cpp`](control_melee_jump.cpp.md).

## State

`SControlMeleeJumpData` — the channel payload: one turn clip per side.

Three tuning constants are declared here rather than authored: a cooldown window of 500 to
1000 milliseconds, a facing check of 165 degrees, and a maximum distance to the enemy of
four world units. The constants are named after the rotation jump they were copied from,
which is worth not reproducing.

Exported units: `reinit`, `activate`, `on_release`, `on_event`, `check_start_conditions` —
the ability lifecycle, nothing else.
