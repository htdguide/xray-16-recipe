# src/xrGame/PHSkeleton.h

> Declares the breakable-skeleton mixin, implemented in [`PHSkeleton.cpp`](PHSkeleton.cpp.md).

**Needs** — [`xrPhysics/PHDefs.h`](../xrPhysics/PHDefs.h.md) · [`PHDestroyableNotificate.h`](PHDestroyableNotificate.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`HangingLamp.cpp`](HangingLamp.cpp.md) · [`HangingLamp.h`](HangingLamp.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`PHDestroyable.cpp`](PHDestroyable.cpp.md) · [`PHSkeleton.cpp`](PHSkeleton.cpp.md) · [`PhysicObject.cpp`](PhysicObject.cpp.md) · [`PhysicObject.h`](PhysicObject.h.md) · [`PhysicsSkeletonObject.cpp`](PhysicsSkeletonObject.cpp.md) · [`PhysicsSkeletonObject.h`](PhysicsSkeletonObject.h.md) · [`helicopter.h`](helicopter.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CPHSkeleton`, mixed into any object whose articulated body may break apart at a
joint. Substance is in [`PHSkeleton.cpp`](PHSkeleton.cpp.md).

Two shape decisions matter before the implementation makes sense.

**It derives from the debris-piece interface.** A breakable skeleton is itself something that
can have been broken off from a larger one, so the same type is both a producer of pieces
and a piece. That is what lets a skeleton break, and then break again.

**A detached body is parked, not delivered.** The field holding pairs of (shell, split bone)
is the asynchrony made visible: the solver detaches a body immediately, the entity to own it
arrives frames later, and the pair sits in the list in between. Any rebuild that can create
an entity synchronously deletes this list and most of the complexity with it.

Exported units:

- `CPHSkeleton` — the mixin.
- `Spawn` — come online as an original, or as a detached piece collecting its body. Returns
  which it was.
- `Update` — per frame: split if the solver fractured, destroy if the removal deadline passed
  and nothing is still parked.
- `Load` — the default debris lifetime, from configuration. Writes a value shared by every
  skeleton in the process.
- `SaveNetState` / `LoadNetState` / `RestoreNetState` — the physical pose on the wire and in
  a save: flags, visible-bone mask, root bone, a bounding box, then per-bone states quantized
  against that box.
- `UnsplitSingle` — hand a parked body and its half of the skeleton to a newly arrived
  entity. The surgery.
- `SetAutoRemove` / `IsRemoving` / `DefaultExitenceTime` — debris ageing, measured in physics
  time so a slowed simulation does not lose its debris.
- `SetNotNeedSave` — debris is not written into a save.
- `RespawnInit` — full reset: whole skeleton visible, root bone zero, nothing parked.
- `InitServerObject` / `CopySpawnInit` — build a piece's server record; decide after a split
  whether the result is now removable litter.
- `SpawnInitPhysics` — demanded of every implementor: build this object's body. The mixin
  knows when, never how.
- `PPhysicsShellHolder` / `PHSkeleton` — the two capability queries by which the rest of the
  physics layer reaches this object.

## Notes

The default debris lifetime is a **static** field, so one value serves every breakable
skeleton in the process and the last object to load wins. Two object types declaring
different removal times cannot both take effect.
