# src/Layers/xrRender/FHierrarhyVisual.h

> Declares the container model: a model that draws nothing itself and holds a list of children that do.

**Needs** — [`FBasicVisual.h`](FBasicVisual.h.md)
**Used by** — [`FHierrarhyVisual.cpp`](FHierrarhyVisual.cpp.md) · [`FLOD.h`](FLOD.h.md) · [`ModelPool.cpp`](ModelPool.cpp.md) · [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md)
**Tier floor** — T2: a list of references and a flag saying who owns them.

## Purpose

Declares the surface implemented in [`FHierrarhyVisual.cpp`](FHierrarhyVisual.cpp.md).

## Exported units

- **`ContainerVisual`** — a model whose substance is its list of child models, plus the flag saying whether it owns them.
- **`sub_model(index)`** — reach one child by position. This is the only structural query on the model hierarchy that reaches outside the renderer; the game layer uses it to address one part of a multi-part model.

**Notes** — The name in the original is misspelled ("Hierrarhy"). It appears in no data file and nothing depends on it.
