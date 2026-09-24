# src/Layers/xrRender/R_Backend_xform.h

> Declares the transform cache that lives inside every command list.

**Needs** — [`R_Backend_xform.cpp`](R_Backend_xform.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`r_constants.h`](r_constants.h.md)
**Used by** — [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`R_Backend_xform.cpp`](R_Backend_xform.cpp.md)
**Tier floor** — T2: a record of matrices and bindings plus its setters.

## Purpose

Declares the surface implemented in [`R_Backend_xform.cpp`](R_Backend_xform.cpp.md) and in the inline half at [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md). The cache holds a back-reference to the command list it belongs to, because publishing a constant is a command-list operation — one transform cache exists per command list, not one globally.

Exported units:

- the transform record itself — three input matrices, four derived, seven constant bindings, one validity flag for the lazy inverse;
- `set_W` / `set_V` / `set_P` — set an input and re-derive what depends on it;
- `get_W` / `get_V` / `get_P` — read back an input, used by code that needs to compose its own matrix against the current camera;
- `set_c_w` / `set_c_invw` / `set_c_v` / `set_c_p` / `set_c_wv` / `set_c_vp` / `set_c_wvp` — bind a name to a location in the current pass *and* immediately publish the current value (defined inline, see [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md));
- `unmap` — drop all seven bindings.

**Notes** — The derived matrices are public rather than accessor-guarded, and other parts of the renderer read them directly to build their own composites (the texture-projection binders do exactly this). That is a real dependency, not an encapsulation slip: a rebuild that hides them must offer the same read access under some name.
