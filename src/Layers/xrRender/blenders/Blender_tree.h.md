# src/Layers/xrRender/blenders/Blender_tree.h

> Declares the foliage template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_tree.cpp`](Blender_tree.cpp.md) · [`Blender_tree_deferred.cpp`](Blender_tree_deferred.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares one template with **two alternative implementations**, exactly one of which is built: [`Blender_tree.cpp`](Blender_tree.cpp.md) for the forward renderer and [`Blender_tree_deferred.cpp`](Blender_tree_deferred.cpp.md) for the deferred ones.

## Exported units

- **`CBlender_Tree`** — the tree class tag at parameter version 1. Detailable, not lightmappable. Two parameters: a blend flag, and a second flag stored as "Object LOD" whose real meaning is *this is not a tree* — it selects the still vertex program instead of the swaying one and drops the alpha reference from 200 to 0, which is how the distant-object impostor system reuses this template.
