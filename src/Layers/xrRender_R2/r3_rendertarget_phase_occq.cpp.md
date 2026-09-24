# src/Layers/xrRender_R2/r3_rendertarget_phase_occq.cpp

> Sets the state that light-volume occlusion queries are drawn under.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it binds targets and depth-stencil state directly.

## Purpose

Between the G-buffer fill and any lighting, every visible light's bounding volume is drawn
once, wrapped in an occlusion query, to learn whether any of its pixels survive the depth
test. This is the state for those draws. No geometry is issued here; the frame graph
issues one volume per light (see [`r2_R_render.cpp`](r2_R_render.cpp.md)).

## `phase_occlusion_queries`

**Contract** — binds the frame's depth buffer with no colour target (or with the back
buffer when multisampling is off, purely because one backend dislikes a bind with no
colour attachment), selects the occlusion description, draws front faces only, restricts
to pixels the G-buffer covered, and turns colour writes off.

```text
FUNCTION phase_occlusion_queries()
  bind: depth = the scene depth; colour = none (or the back buffer)
  material = the occlusion description
  cull back faces
  stencil: pass where the stored value is at least 1
  colour writes off
```

**Invariants** — depth *writes* must already be off, which the occlusion description
carries; a query that wrote depth would corrupt the G-buffer's depth for every later pass.

**Notes** — restricting to stencil at least one means a light volume floating over empty
sky is reported invisible even though nothing occludes it, which is exactly right: with no
surface there, it would light nothing. Front faces only is enough because the query asks
"did anything pass", not "how much" — and the front faces of a convex volume are the
nearest surface of it.
