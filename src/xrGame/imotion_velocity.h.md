# src/xrGame/imotion_velocity.h

> Declares the death-animation variant that drives the body by setting velocities rather than positions.

**Needs** — [`interactive_motion.h`](interactive_motion.h.md) · [`imotion_velocity.cpp`](imotion_velocity.cpp.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`imotion_velocity.cpp`](imotion_velocity.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`imotion_velocity.cpp`](imotion_velocity.cpp.md): the
four hooks of [`interactive_motion.h`](interactive_motion.h.md) — start, end, collide, move —
filled in for the velocity-driven variant.

Exported units: `imotion_velocity`, overriding `state_start`, `state_end`, `collide` and
`move_update`. None is public; the base drives all four.
