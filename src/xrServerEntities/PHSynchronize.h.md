# src/xrServerEntities/PHSynchronize.h

> The interface anything physically simulated must satisfy so that its state can be captured into an entity record and pushed back out of one.

**Needs** — [`PHNetState.h`](PHNetState.h.md)
**Used by** — [`PHSkeleton.cpp`](../xrGame/PHSkeleton.cpp.md) · [`PhysicObject.cpp`](../xrGame/PhysicObject.cpp.md) · [`PhysicsSkeletonObject.cpp`](../xrGame/PhysicsSkeletonObject.cpp.md) · [`actor_defs.h`](../xrGame/actor_defs.h.md) · [`PHActivationShape.cpp`](../xrPhysics/PHActivationShape.cpp.md) · [`PHCharacter.h`](../xrPhysics/PHCharacter.h.md) · [`PHElement.h`](../xrPhysics/PHElement.h.md) · [`PHElementNetState.cpp`](../xrPhysics/PHElementNetState.cpp.md) · [`PHShellNetState.cpp`](../xrPhysics/PHShellNetState.cpp.md) · [`PHWorld.cpp`](../xrPhysics/PHWorld.cpp.md) · [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md)
**Tier floor** — T2: four operations, two of which are coordinate-frame conversions.

## Purpose

This is the seam between the entity records in this chapter and the rigid-body world in
chapter 16. A record that has physics does not know what a body is; it knows it can ask its
physics side for a snapshot and hand one back. Four operations express that, and the two
that are not obvious are the interesting ones.

## The contract an implementor satisfies

**`capture(out snapshot)`** — fill a snapshot with this object's current body state. Must
be callable at any time; the caller decides whether to serialize it, quantize it or compare
it.

**`restore(snapshot)`** — set this object's body state from a snapshot. The object is
expected to be in a valid but arbitrary state beforehand; afterwards it is exactly the
snapshot. This is what a save load and a network correction both go through.

**`export_to(packet)` / `import_from(packet)`** — the object's own network serialization,
for objects that want something other than a plain snapshot. Default to writing nothing, so
an object that is fully described by its snapshot need not implement them.

**`body_transform(orientation, position) -> transform`** — build the object's world
transform from a snapshot's orientation and position. Not a trivial compose: a physics body's
frame is the body's *centre of mass*, while the object's frame is its visual origin, and the
offset between them is the object's own. This is why the conversion is a demand on the
implementor rather than a shared helper.

**`bone_transform(orientation, position) -> transform`** — the same for one bone of a
skeleton, where the offset is the bone's bind pose rather than the object's centre of mass.

## Notes

The two conversions being *virtual demands* rather than a function is the load-bearing part
of this interface. It means the records and the network layer can move body state around in
one universal shape (the snapshot, which is pure position and orientation) while each kind
of object keeps its own idea of where its origin is relative to its mass. A rebuild that
flattens this into one shared conversion will find ragdolls and vehicles offset from their
visuals.
