# src/xrPhysics/GeometryBits.h

> Declares the three calls that place a shape in a collision category.

**Needs** — [`GeometryBits.cpp`](GeometryBits.cpp.md)
**Used by** — [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`GeometryBits.cpp`](GeometryBits.cpp.md) · [`PHWorld.cpp`](PHWorld.cpp.md)
**Tier floor** — T3: three static calls.

## Purpose

Declares the surface implemented in [`GeometryBits.cpp`](GeometryBits.cpp.md).

## Exported units

- **initialize a primitive shape** — currently does nothing; the hook exists so category
  assignment has one place to live.
- **initialize the level's static mesh shape** — puts it in the static category.
- **make a shape ignore static geometry** — clears the static category from what that shape
  will collide with.
