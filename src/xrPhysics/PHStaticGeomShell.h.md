# src/xrPhysics/PHStaticGeomShell.h

> Declares a collision volume with no body: something the world can be stopped by, which is never itself pushed.

**Needs** — [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md) · [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md) · [`IPHStaticGeomShell.h`](IPHStaticGeomShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`IPHStaticGeomShell.h`](IPHStaticGeomShell.h.md) · [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md)
**Tier floor** — T1: it owns solver shapes and participates in the world's collision phase.

## Purpose

Declares the surface implemented in [`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md); the
interface it satisfies is [`IPHStaticGeomShell.h`](IPHStaticGeomShell.h.md).

A shell ([`PHShell.h`](PHShell.h.md)) is bodies and joints. This is the degenerate case with
neither: shapes, a placement, and nothing that moves. It exists because a large class of game
objects need to be *collidable* without being *simulated* — an intact door before it is broken, a
crate that has not been knocked over yet, a ladder — and paying for a body, an island and a step for
each of them would be a real cost on a level with hundreds.

## Exported units

- **`Activate(form)`** — build the shapes, place them, and register spatially.
- **`Deactivate`** — the reverse.
- **`PhDataUpdate`** — the one-shot step; contract in the implementation twin.
- **`EnableObject`** — being touched re-registers this object for exactly one update.
- **`PhTune`**, **`InitContact`** — empty: a static volume adds nothing to a contact and needs no
  pre-solve pass.
- **`get_elements_number`**, **`get_element_sync`** — zero and nothing: there is no per-element state
  to synchronize, because there are no elements.

Shape composition — adding boxes, spheres, meshes, setting materials and contact callbacks — comes
from [`PHGeometryOwner.h`](PHGeometryOwner.h.md) unchanged.

## Notes

The type reports itself as a physics object to the spatial index, so queries that look for physics
find it. That is the whole point: a static geometry shell must be indistinguishable from a real
shell to everything that asks the world what is where.
