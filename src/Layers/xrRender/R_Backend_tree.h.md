# src/Layers/xrRender/R_Backend_tree.h

> Declares the wind/vegetation constant channel that lives inside every command list.

**Needs** — [`R_Backend_tree.cpp`](R_Backend_tree.cpp.md) · [`R_Backend.h`](R_Backend.h.md)
**Used by** — [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) · [`FTreeVisual.cpp`](FTreeVisual.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_tree.cpp`](R_Backend_tree.cpp.md)
**Tier floor** — T3: eight bindings and their setters.

## Purpose

Declares the surface implemented in [`R_Backend_tree.cpp`](R_Backend_tree.cpp.md). One instance per command list, holding a back-reference to it.

Exported units:

- the eight constant bindings for the frozen shader names `m_xform_v`, `m_xform`, `consts`, `wave`, `wind`, `c_scale`, `c_bias`, `c_sun`;
- `set_c_*` — one per binding, recording where the name lives in the current pass;
- `set_m_xform_v` / `set_m_xform` / `set_consts` / `set_wave` / `set_wind` / `set_c_scale` / `set_c_bias` / `set_c_sun` — publish a value into a binding, no-op when unbound;
- `unmap` — drop all eight bindings.

**Notes** — The binding setters do not publish a current value, because this cache stores none: every value comes from the vegetation visual at draw time. Exactly one visual kind writes through this cache — the wind-animated tree, in both its static and progressive-mesh forms — so the module could equally have been a member of that visual. It lives on the command list because the *bindings* are pass state and pass state belongs to the command list.
