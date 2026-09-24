# src/xrGame/physics_shell_animated.h

> Declares the animation-driven rigid-body assembly: a physics body that follows a pose instead of being solved for one. Implemented in [`physics_shell_animated.cpp`](physics_shell_animated.cpp.md).

**Needs** — [`physics_shell_animated.cpp`](physics_shell_animated.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`interactive_animation.cpp`](interactive_animation.cpp.md) · [`interactive_animation.h`](interactive_animation.h.md) · [`physics_shell_animated.cpp`](physics_shell_animated.cpp.md)
**Tier floor** — T2: a declaration over a physics handle

## Purpose

Declares `physics_shell_animated`. An ordinary rigid-body assembly is *solved*: the physics
world decides where its parts go and the skeleton is posed from the result. This one is the
inverse — the animation decides the pose and the bodies are dragged to match, so that the
assembly is present in the world as a collider and a velocity source without ever being
simulated.

The owner chooses at construction whether velocities are also derived from the animation,
which is the single option this type carries.

Exported units:

- `physics_shell_animated` — construct from a holder, with or without velocity derivation.
- `shell` — the underlying assembly, read-only and mutable forms.
- `update` — drive the assembly from a world transform for one frame.
- `create_shell` — the construction step, overridable so a subclass can build a different
  assembly.
