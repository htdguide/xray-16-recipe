# src/xrServerEntities/PHNetState.h

> Declares the rigid-body snapshot that physics objects put in their entity record and on the wire, and the multi-bone version a ragdoll uses.

**Needs** — [`PHNetState.cpp`](PHNetState.cpp.md) · [`xrCore/_vector3d.h`](../xrCore/_vector3d.h.md) · [`xrCore/_quaternion.h`](../xrCore/_quaternion.h.md)
**Used by** — [`PHNetState.cpp`](PHNetState.cpp.md) · [`PHSynchronize.h`](PHSynchronize.h.md) · [`xrServer_Objects.cpp`](xrServer_Objects.cpp.md) · [`xrServer_Objects.h`](xrServer_Objects.h.md)
**Tier floor** — T1: a fixed-layout snapshot written to saves and packets at chosen widths.

## Purpose

Declares the surface implemented in [`PHNetState.cpp`](PHNetState.cpp.md).

## Exported units

- **body snapshot** — one rigid body's state: linear and angular velocity, force, torque,
  position, previous position, orientation, previous orientation, and whether the body is
  awake. Two of those fields share storage with an alternative pair (acceleration and
  maximum velocity) used by bodies that are driven rather than simulated.
- **skeleton snapshot** — a ragdoll's state: a bone mask, the root bone, a bounding box, and
  one body snapshot per bone.
- The six read/write entry points per snapshot: export/import (network), save/load
  (persistence), and a quantized save/load pair that takes an explicit bounding box.

## Notes

The union between the orientation and the (acceleration, maximum velocity) pair is a memory
economy from a 32-bit era: a body is either simulated (and has an orientation) or driven
(and has a target velocity), never both. A rebuild may simply carry both — but must know
which of the two the serialization is writing, because the bytes are the same either way and
nothing in the stream distinguishes them. The kind of body decides, and the kind comes from
the entity's class.
