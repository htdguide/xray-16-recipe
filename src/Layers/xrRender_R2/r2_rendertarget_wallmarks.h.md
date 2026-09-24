# src/Layers/xrRender_R2/r2_rendertarget_wallmarks.h

> Declares a free-standing wallmark phase that nothing defines or calls.

**Needs** — nothing
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: one declaration.

## Purpose

A leftover. The wallmark phase that the frame graph actually uses is a method on the
render-target object, supplied by each backend
(see [`r2_R_render.cpp`](r2_R_render.cpp.md) for where it is called and the backend
chapters for what it does). This header declares a same-named free function that has no
definition anywhere in the tree and no caller. A rebuild should not create it.

## `phase_wallmarks`

**Contract** — declared, never defined, never called. Recorded here only so the mirror is
complete and so a reader who greps the name is not misled into thinking there are two
wallmark paths.
