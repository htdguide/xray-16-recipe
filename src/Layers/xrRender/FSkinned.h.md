# src/Layers/xrRender/FSkinned.h

> Declares the skinned model: one mesh bound to one skeleton, in a still and a simplifying variant, plus the four skeleton-aware queries the game layer needs.

**Needs** — [`FVisual.h`](FVisual.h.md) · [`FProgressive.h`](FProgressive.h.md) · [`SkeletonX.h`](SkeletonX.h.md) · [`FSkinnedTypes.h`](FSkinnedTypes.h.md)
**Used by** — [`FSkinned.cpp`](FSkinned.cpp.md) · [`ModelPool.cpp`](ModelPool.cpp.md)
**Tier floor** — T1: it is a device-buffer model plus per-vertex bone packing.

## Purpose

Declares the surface implemented in [`FSkinned.cpp`](FSkinned.cpp.md). A skinned model is a *child* of a skeleton, not a thing in its own right: the skeleton ([`SkeletonCustom.cpp`](SkeletonCustom.cpp.md)) is a container model whose children are these, and each child is one material's worth of the creature's surface.

## Exported units

- **`SkinnedExtension`** — the shared half: building the device vertex buffer from the model file's skinning data, collecting each bone's face list, and the three per-bone queries.
- **`SkinnedVisual_Static`** — a skinned child that draws its whole mesh.
- **`SkinnedVisual_Progressive`** — a skinned child that draws a slide-window range chosen by level of detail.

The four operations that distinguish a skinned child from an ordinary model:

- **`after_load(skeleton, child_index)`** — bind to the parent skeleton and build the per-bone face lists. Called once, after both the skeleton and this child exist.
- **`pick_bone(...)`** — intersect a ray against one bone's faces, in their *current animated* positions.
- **`fill_vertices(...)`** — cut a decal into one bone's faces.
- **`enumerate_bone_vertices(...)`** — hand every vertex of one bone's faces to a caller.

**Notes** — Both concrete classes inherit from an ordinary model type *and* from the skinning extension. That is how one mesh gets both the draw path and the skeleton binding. The duplication between the two classes is near total — each is the other with a different draw and a different index range — and a rebuild should carry the level-of-detail range as data rather than as a type.
