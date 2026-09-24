# src/xrPhysics/tri-colliderknoopc/dTriCollideK.h

> One include for all three primitive-versus-triangle cases.

**Needs** — [`dTriBox.h`](dTriBox.h.md) · [`dTriSphere.h`](dTriSphere.h.md) · [`dTriCylinder.h`](dTriCylinder.h.md)
**Used by** — [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) · [`dTriCallideK.cpp`](dTriCallideK.cpp.md) · [`dTriList.cpp`](dTriList.cpp.md) · [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md)
**Tier floor** — T4: an umbrella.

## Purpose

Names the three primitives the mesh collider supports — box, sphere, cylinder — by pulling in
their headers together. Callers that need one usually need all three, because the traversal is
instantiated for each.

The list is the only content: **a triangle mesh collides against exactly these three shapes**
and nothing else. There is no capsule case and no mesh-versus-mesh case, which is why every
character in the game is a cylinder
([`../dcylinder/dCylinder.h`](../dcylinder/dCylinder.h.md)) and why static geometry never
moves.

## Stateless.
