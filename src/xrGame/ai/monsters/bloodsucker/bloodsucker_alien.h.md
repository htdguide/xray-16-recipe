# src/xrGame/ai/monsters/bloodsucker/bloodsucker_alien.h

> Declares the camera takeover: while it is active the player sees through the bloodsucker's eyes and cannot fight back.

**Needs** — [`bloodsucker_alien.cpp`](bloodsucker_alien.cpp.md)
**Used by** — [`bloodsucker.h`](bloodsucker.h.md) · [`bloodsucker_alien.cpp`](bloodsucker_alien.cpp.md)
**Tier floor** — T2: two effectors on the player's camera chain with an active flag

## Purpose

Declares the surface implemented in [`bloodsucker_alien.cpp`](bloodsucker_alien.cpp.md).

## Exported units

- **the takeover** — holds its creature, the two effectors it installs, the saved crosshair
  state, and an active flag.
- **activate / deactivate / is active** — idempotent in both directions.
- the two effector classes are private to the implementation.
