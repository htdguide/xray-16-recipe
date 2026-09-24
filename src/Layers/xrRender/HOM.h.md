# src/Layers/xrRender/HOM.h

> Declares the software occlusion map: an authored set of blocking surfaces, rasterized into a depth hierarchy each frame, against which anything with a bounding volume can be tested.

**Needs** — [`occRasterizer.h`](occRasterizer.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`DetailManager.cpp`](DetailManager.cpp.md) · [`HOM.cpp`](HOM.cpp.md) · [`occRasterizer.h`](occRasterizer.h.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__sector.h`](r__sector.h.md) · [`r__sector_traversal.cpp`](r__sector_traversal.cpp.md)
**Tier floor** — T1: it owns a tree over a triangle soup and a software rasterizer's buffer.

## Purpose

Declares the surface implemented in [`HOM.cpp`](HOM.cpp.md).

## Exported units

- **`OcclusionMap`** — the whole mechanism: load and unload the level's occluder set, enable and disable, dispatch this frame's rasterization onto a worker, and four visibility queries.
- **`visible(visibility record)`** — the main query, for scene objects. Carries its own temporal caching; see the implementation.
- **`visible(world box)`** — the same test without caching.
- **`visible(polygon)`** — for a portal's clipped outline.
- **`visible(screen rectangle, depth)`** — for something already in screen space, such as a light's screen extent.
- **`dispatch_rasterization()`** — build this frame's occlusion map on a worker; returns the task to wait on.

**Notes** — In a checked build the class also registers a per-frame debug draw that paints the occluder geometry over the scene. The interface therefore has a different shape in the two builds, which is the binary-compatibility hazard the chapter-4 README warns about — here it is harmless, because nothing loads this across a module boundary.
