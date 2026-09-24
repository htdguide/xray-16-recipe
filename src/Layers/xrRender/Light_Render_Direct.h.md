# src/Layers/xrRender/Light_Render_Direct.h

> A vestige: declares one operation for computing a spot light's transform, with no implementation anywhere.

**Needs** — [`light.h`](light.h.md)
**Used by** — [`Light_Render_Direct.cpp`](Light_Render_Direct.cpp.md) · [`Light_Render_Direct_ComputeXFS.cpp`](Light_Render_Direct_ComputeXFS.cpp.md)
**Tier floor** — T2.

## Purpose

Declares a single-method class whose method has no definition in the repository. [`Light_Render_Direct.cpp`](Light_Render_Direct.cpp.md) contains nothing but the include of this header.

## `compute_spot_transform(light)`

**Contract** — none discoverable. By its name it would build a spot light's projection transform and the derived visibility information, which is work that is done today inside the light record itself (see [`light.cpp`](light.cpp.md)) and inside the shadow allocation.

**Notes** — Nothing calls it and nothing defines it; a call site would fail to link. This is a removed subsystem's leftover declaration. Both files are recorded here only because the mirror must be complete; a rebuild writes neither.

The sibling file [`Light_Render_Direct_ComputeXFS.cpp`](Light_Render_Direct_ComputeXFS.cpp.md) has a matching name and is the place the implementation presumably once lived.
