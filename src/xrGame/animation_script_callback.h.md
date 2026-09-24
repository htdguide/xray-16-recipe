# src/xrGame/animation_script_callback.h

> Declares the one-slot animation-end mailbox implemented in [`animation_script_callback.cpp`](animation_script_callback.cpp.md).

**Needs** — [`animation_script_callback.cpp`](animation_script_callback.cpp.md)
**Used by** — [`PhysicObject.cpp`](PhysicObject.cpp.md) · [`PhysicObject.h`](PhysicObject.h.md) · [`animation_script_callback.cpp`](animation_script_callback.cpp.md)
**Tier floor** — T3: three flags and two methods

## Purpose

Declares the surface implemented in
[`animation_script_callback.cpp`](animation_script_callback.cpp.md). Small enough to be
embedded by value in whatever game object wants script animation callbacks.

Exported units:

- **`play_cycle`** — play a named motion and arm the end callback if the motion has an end.
- **`update`** — drain the latch into the object's script callback.
