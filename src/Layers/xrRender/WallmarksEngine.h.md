# src/Layers/xrRender/WallmarksEngine.h

> Declares the decal store and its two submission paths.

**Needs** — [`Shader.h`](Shader.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`xrEngine/Render.h`](../../xrEngine/Render.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`WallmarksEngine.cpp`](WallmarksEngine.cpp.md)
**Tier floor** — T2: a declaration, but it fixes the vertex layout decals are written in.

## Purpose

Declares the store implemented in [`WallmarksEngine.cpp`](WallmarksEngine.cpp.md), and names the one record other modules see: a static wallmark is a bounding sphere, a list of lit-and-textured vertices, and a remaining lifetime.

## Exported units

- **`CWallmarksEngine`** — the store. Holds one slot per material, a pool of recycled static records, one shared geometry handle, and the scratch state the cut uses (the clipper, the polygon buffers, the collision query object, the adjacency table) as members rather than locals, because the cut runs on the physics path and must not allocate.
- **`static_wallmark`** — bounding sphere, vertex list, time to live.
- **`AddStaticWallmark`** — cut a decal out of the static world at a contact point.
- **`AddSkeletonWallmark`** (two forms) — ask an animated model to cut one into its own skin, or file one the model already cut.
- **`Render`** — draw every slot, age the static marks, discard the skeleton ones.
- **`clear`** — drop everything, on level unload.

The scratch members are the file's one real statement: every one of them exists so that `AddStaticWallmark` can run from the physics thread without touching the allocator, at the cost of making the whole cut single-threaded behind one lock.
