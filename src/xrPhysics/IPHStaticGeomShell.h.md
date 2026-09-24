# src/xrPhysics/IPHStaticGeomShell.h

> Builds a body-less collision presence for an object that never moves but must
> still be felt.

**Needs** — [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md) · [`IClimableObject.h`](IClimableObject.h.md)
**Used by** — [`BreakableObject.cpp`](../xrGame/BreakableObject.cpp.md) · [`ClimableObject.cpp`](../xrGame/ClimableObject.cpp.md) · [`PHStaticGeomShell.h`](PHStaticGeomShell.h.md)
**Tier floor** — T3: three constructors and a destructor.

## Purpose

Most static collision comes from the level's compiled triangle soup
([Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)).
Some does not: an object placed at run time that will never move — and, specifically, a
**ladder** — needs a collision shape in the physics world with no body behind it. This
header is the factory for those.

The interface itself is empty, which is honest: a static geometry shell has no behaviour to
expose. It exists only so that the creator can later destroy what it made without knowing
the concrete type. The substance is in
[`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md).

## `P_BuildStaticGeomShell`

**Contract** — builds collision geometry for a shell holder from the shapes already
described in its skeleton, attaches the given contact callback, and returns the handle. The
geometry does not move and is never integrated; it participates only in collision detection.

## `P_BuildLeaderGeomShell`

**Contract** — the ladder case. Takes a climbable object and an oriented box, and builds a
single static box geometry from that box with the given contact callback. Separate from the
general builder because the caller here is not a shell holder at all and has no skeleton to
read shapes from — the box *is* the whole description.

**Notes** — the ladder's contact callback is what actually makes climbing work: the callback
recognises a character touching the ladder volume and hands the ladder to that character's
climbing state machine ([`ElevatorState.cpp`](ElevatorState.cpp.md)). The collision itself is
usually suppressed. A rebuilder should read "static geom shell" as *trigger volume that
happens to be built out of collision machinery*.

## `DestroyStaticGeomShell`

**Contract** — removes the geometry from the world and frees it, clearing the caller's
handle. Safe on an already-cleared handle.
