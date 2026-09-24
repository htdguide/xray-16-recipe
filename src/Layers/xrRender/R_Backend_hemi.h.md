# src/Layers/xrRender/R_Backend_hemi.h

> Declares the per-object lighting and per-visual data channel that lives inside every command list.

**Needs** — [`R_Backend_hemi.cpp`](R_Backend_hemi.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`RenderVisual.h`](../../Include/xrRender/RenderVisual.h.md)
**Used by** — [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md) · [`R_Backend_hemi.cpp`](R_Backend_hemi.cpp.md)
**Tier floor** — T3: six bindings and their setters.

## Purpose

Declares the surface implemented in [`R_Backend_hemi.cpp`](R_Backend_hemi.cpp.md). Like the transform cache it holds a back-reference to its owning command list, so one exists per command list.

Exported units:

- the six constant bindings, public and written directly by the standard constant binders as well as through setters — the three per-visual data slots have no setter at all and are assigned by name from the binder table;
- `set_c_pos_faces` / `set_c_neg_faces` / `set_c_material` — bind a name to a location;
- `set_pos_faces` / `set_neg_faces` — publish the two halves of the ambient cube;
- `set_material` — publish the ambient term, sun term and material coordinate;
- `c_update` — publish a visual's camouflage, custom and entity data;
- `unmap` — drop all six bindings.

**Notes** — Unlike the transform cache, the binding setters here do *not* immediately publish the current value: there is no current value to publish, because everything this module sends is supplied by the caller at draw time. That asymmetry between the two caches is deliberate and a rebuild should preserve it rather than making the two look alike.
