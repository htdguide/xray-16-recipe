# src/xrGame/wallmark_manager.h

> Declares the burst-decal placer: a point, a pool of decal appearances, and the sweep that marks everything around it.

**Needs** — [`wallmark_manager.cpp`](wallmark_manager.cpp.md) · [`Include/xrRender/FactoryPtr.h`](../Include/xrRender/FactoryPtr.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Explosive.cpp`](Explosive.cpp.md) · [`Explosive.h`](Explosive.h.md) · [`wallmark_manager.cpp`](wallmark_manager.cpp.md)
**Tier floor** — T2: holds a renderer-owned handle and a position

## Purpose

Declares the surface implemented in
[`wallmark_manager.cpp`](wallmark_manager.cpp.md).

Exported units:

- `PlaceWallmarks(centre)` — the public entry: load the appearance pool and sweep.
- `StartWorkflow` — the sweep itself, exposed because it was once scheduled separately.
- `AddWallmark(direction, start, range, size, appearances, triangle)` — place one decal
  along a ray, gated on the struck material accepting blood.
- `Load(section)` — append the named section's decal appearances to the pool.
- `Clear` — empty the pool.
- `m_owner` — the object excluded from the sweep's occlusion rays; written directly by
  whoever constructs the manager.

## State

See [`wallmark_manager.cpp`](wallmark_manager.cpp.md).

**Notes** — The decal-appearance pool is held through the renderer's factory indirection
rather than as a concrete type, because what a decal appearance *is* differs between the
shipped render backends and the game layer must not know. A rebuild keeps the seam: the
game names appearances by string and hands the renderer a triangle and a point.

The class name is misspelled in the original (one `l` in *wallmark*). Nothing depends on it.
