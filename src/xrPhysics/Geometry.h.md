# src/xrPhysics/Geometry.h

> Declares the wrapper that gives every primitive collision shape a uniform
> lifecycle, mass, local placement and extent query.

**Needs** — [`Geometry.cpp`](Geometry.cpp.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`xrEngine/IPhysicsGeometry.h`](../xrEngine/IPhysicsGeometry.h.md)
**Used by** — [`PHCollisionDamageReceiver.cpp`](../xrGame/PHCollisionDamageReceiver.cpp.md) · [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`CalculateTriangle.h`](CalculateTriangle.h.md) · [`Geometry.cpp`](Geometry.cpp.md) · [`GeometryBits.cpp`](GeometryBits.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHGeometryOwner.cpp`](PHGeometryOwner.cpp.md) · [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHMoveStorage.cpp`](PHMoveStorage.cpp.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md)
**Tier floor** — T1: it manages shapes the dynamics library owns and computes mass tensors
the library will consume.

## Purpose

Declares the shape surface implemented in [`Geometry.cpp`](Geometry.cpp.md): one abstract
wrapper and three concretions — box, sphere, cylinder — matching the shape kinds a
skeleton's collision data can describe.

## Exported units

- **`CODEGeom`** — the abstract wrapper. Owns one shape plus its transform wrapper, the
  bone it follows, and its shape flags. Contracts in
  [`Geometry.cpp`](Geometry.cpp.md).
- **`CBoxGeom`** — an oriented box, from a half-size plus rotation plus offset. The only
  shape whose size can be changed after creation, which the camera-collision path needs.
- **`CSphereGeom`** — a sphere, from a centre and radius.
- **`CCylinderGeom`** — a cylinder, from a centre, axis direction, height and radius; the
  radius can be changed after creation.
- **shape extent query** — the projection of a shape onto an arbitrary axis, as a low and
  high bound, implemented separately per shape kind and used to fit a box around anything.

## Notes

`CODEGeom` implements the engine's own geometry interface, so the renderer's debug views and
the game's bone-hit tests can ask a physical shape for its transform and box without
depending on this module. That, and not polymorphism over shapes, is why the abstract base
exists at all — the three concrete shapes share very little code.

The matrix-multiply macro at the top of the file transposes one operand while multiplying,
because the engine's matrices and the solver's matrices disagree on row-versus-column order.
It is incidental; a rebuild picks one convention and converts at the boundary once.
