# src/xrGame/EffectorFall.h

> Declares the landing dip and the timed depth-of-field override implemented in [`EffectorFall.cpp`](EffectorFall.cpp.md).

**Needs** — [`xrEngine/Effector.h`](../xrEngine/Effector.h.md)
**Used by** — [`EffectorFall.cpp`](EffectorFall.cpp.md) · [`Spectator.cpp`](Spectator.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares two self-retiring camera-chain effectors. Substance in
[`EffectorFall.cpp`](EffectorFall.cpp.md).

Exported units:

- `CEffectorFall` — constructed with a landing severity and lifetime; dips the camera
  through one half-sine and expires. Excludes the first-person weapon model.
- `CEffectorDOF` — constructed with a depth-of-field triple and a duration; pushes the
  parameters at construction, restores them at the deadline, and never moves the camera.
