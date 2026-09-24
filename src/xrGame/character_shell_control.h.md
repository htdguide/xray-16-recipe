# src/xrGame/character_shell_control.h

> Declares the death-ragdoll tuning implemented in [`character_shell_control.cpp`](character_shell_control.cpp.md).

**Needs** — [`character_shell_control.cpp`](character_shell_control.cpp.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`character_shell_control.cpp`](character_shell_control.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`character_shell_control.cpp`](character_shell_control.cpp.md). Embedded by value in a
living entity's physics support; it is pure tuning state with no identity of its own.

Exported units:

- **`Load`** — read the ten required and two optional tuning values from the creature's
  section.
- **`set_kill_hit`, `set_fatal_impulse`** — shape the killing hit's direction and impulse.
- **`apply_start_velocity_factor`** — scale the ragdoll's launch velocity, with anomalies
  treated differently.
- **`set_start_shell_params`** — air resistance and the contact callback, applied once when
  the ragdoll is created.
- **`TestForWounded`** — decide once, at death, whether the character was already lying down.
- **`CalculateTimeDelta`** — the shared wall-clock delta the ramps run on.
- **`UpdateFrictionAndJointResistanse`** — advance the ramps and apply them.
- **`curr_skin_friction_in_death`** — the value the solver's contact callback reads.
