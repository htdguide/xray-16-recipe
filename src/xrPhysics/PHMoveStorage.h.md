# src/xrPhysics/PHMoveStorage.h

> Declares the set of shapes whose motion between steps is swept, so a fast small object cannot pass through a thin obstacle.

**Needs** — [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHMoveStorage.cpp`](PHMoveStorage.cpp.md)
**Used by** — [`PHMoveStorage.cpp`](PHMoveStorage.cpp.md) · [`PHObject.cpp`](PHObject.cpp.md) · [`PHShell.h`](PHShell.h.md)
**Tier floor** — T2: a collection and a pair of positions; the position extraction in the implementation is T1.

## Purpose

An object moving faster than its own size per fixed step can begin a step on one side of a wall and
end it on the other, with the collider finding nothing at either endpoint. The module's answer is
to collide the *segment* the object travelled rather than its endpoints — see the swept-motion path
in [`PHObject.cpp`](PHObject.cpp.md).

This type is the opt-in list: the shapes for which that is worth doing. It is a list rather than a
global setting because the sweep costs a ray query per shape per step, which is not worth paying
for a wall or a resting crate.

## Exported units

- `CPHMoveStorage` — the collection. `add(shape)`, `clear()`, `empty()`, and iteration.
- `CPHPositionsPairs` — the iterator. Beyond advancing, it answers two questions about the shape it
  points at: the pair of world positions bounding this step's motion (`positions`), and the shape
  handle to present to the collider (`shape`).

## Notes

A shell enables the sweep by adding shapes here and turning on its motion-tracing flag; adding no
shapes and turning the flag on is harmless but pointless, which is why the flag is derived from
emptiness rather than set independently (`enable_geom_trace` in [`PHShell.cpp`](PHShell.cpp.md)).

The iterator, not the storage, knows how to extract positions, because the answer depends on
whether the shape is attached to its body directly or through an offset wrapper. See
[`PHMoveStorage.cpp`](PHMoveStorage.cpp.md).
