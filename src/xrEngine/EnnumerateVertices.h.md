# src/xrEngine/EnnumerateVertices.h

> A callback shape for walking a mesh's vertices without materializing them.

**Needs** — [`xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — [`IKFoot.cpp`](../xrGame/IKFoot.cpp.md)
**Tier floor** — T3: one callback signature.

## Purpose

Several subsystems need every vertex of a collision shape or a renderable — to build a bounding volume, to scatter particles over a surface, to test a volume. Each of those meshes is stored differently, so the traversal is the owner's and the consumer supplies a per-vertex callback. Declaring the callback shape here, in the engine, is what lets a physics shape hand vertices to a renderer helper without either knowing the other.

## `VertexVisitor`

**Contract** — Invoked once per vertex with a world-space position. No return value: a visitor cannot stop the walk early, which means a consumer that wants an early exit must latch a flag and ignore the remainder. The walk is synchronous and the position is valid only for the duration of the call.
