# src/Layers/xrRender/blenders/uber_deffer.h

> Declares the two shared g-buffer pass builders.

**Needs** — [`uber_deffer.cpp`](uber_deffer.cpp.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_BmmD_deferred.cpp`](Blender_BmmD_deferred.cpp.md) · [`Blender_Model_EbB_deferred.cpp`](Blender_Model_EbB_deferred.cpp.md) · [`Blender_detail_still_deferred.cpp`](Blender_detail_still_deferred.cpp.md) · [`Blender_tree_deferred.cpp`](Blender_tree_deferred.cpp.md) · [`blender_deffer_aref.cpp`](blender_deffer_aref.cpp.md) · [`blender_deffer_flat.cpp`](blender_deffer_flat.cpp.md) · [`blender_deffer_model.cpp`](blender_deffer_model.cpp.md) · [`uber_deffer.cpp`](uber_deffer.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`uber_deffer.cpp`](uber_deffer.cpp.md).

Exported units:

- **`uber_deffer`** — build one g-buffer pass from the material's texture names. Takes the compile context, a high-quality flag, a vertex and a pixel *specification* word (the family, e.g. `"model"`, `"base"`, `"impl"`), an alpha-test flag, an optional detail-texture override, and a flag that leaves the pass open so the caller can append state.
- **`uber_shadow`** — the same derivation reduced to a depth-only pass; exists only where hardware tessellation does.
