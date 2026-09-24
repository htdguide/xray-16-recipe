# src/Layers/xrRender/FLOD.h

> Declares the far-distance impostor: eight authored billboards of one object, one per compass direction, and the screen-coverage factor that says when to switch to it.

**Needs** — [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md)
**Used by** — [`FLOD.cpp`](FLOD.cpp.md) · [`ModelPool.cpp`](ModelPool.cpp.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) · [`r__dsgraph_render_lods.cpp`](r__dsgraph_render_lods.cpp.md)
**Tier floor** — T1: the facet record is read as a byte image from a shipped model file and the draw record is an exact vertex layout.

## Purpose

Declares the surface implemented in [`FLOD.cpp`](FLOD.cpp.md).

## Exported units

- **`ImpostorVisual`** — a container model that also carries eight authored quads and a level-of-detail factor.
- **`ImpostorVertex`** — one corner of one authored quad: position, texture coordinates, packed colour with hemisphere term, and a sun term.
- **`ImpostorFacet`** — one quad's four corners plus the outward normal computed at load.
- **`ImpostorDrawVertex`** — the drawn form: **two** of everything, because the draw blends between two adjacent facets.

**Notes** — The drawn vertex record is declared twice, once here and once again inside the implementation file. The two are identical. It is a duplication, not a difference.
