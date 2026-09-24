# src/xrGame/PhysicsSkeletonObject.h

> Declares the jointed breakable prop implemented in [`PhysicsSkeletonObject.cpp`](PhysicsSkeletonObject.cpp.md).

**Needs** — [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md)
**Used by** — [`PhysicObject.h`](PhysicObject.h.md) · [`PhysicsSkeletonObject.cpp`](PhysicsSkeletonObject.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CPhysicsSkeletonObject` as the join of the physics-shell holder and the
breakable-skeleton mixin, and names the two accessors each base needs to reach the other
(the mixin asks its host for the shell holder; the holder asks for the mixin). Substance
is in [`PhysicsSkeletonObject.cpp`](PhysicsSkeletonObject.cpp.md).

Exported units:

- `CPhysicsSkeletonObject` — the prop.
- Lifecycle: `net_Spawn`, `net_Destroy`, `Load`, `UpdateCL`, `shedule_Update`, `net_Save`,
  `net_SaveRelevant`, `UsedAI_Locations`.
- Physics hooks: `SpawnInitPhysics`, `CreatePhysicsShell`, `PHObjectPositionUpdate`.

## Notes

The two "which base am I" accessors exist only because the language forces a diamond to be
resolved by hand. In a rebuild where the two halves are components on one entity rather
than base classes, both vanish.
