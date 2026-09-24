# src/Layers/xrRender/FVisual.h

> Declares the plain static model: one mesh, one material, one draw call — optionally with a stripped-down twin for depth-only passes.

**Needs** — [`FBasicVisual.h`](FBasicVisual.h.md)
**Used by** — [`FProgressive.h`](FProgressive.h.md) · [`FSkinned.h`](FSkinned.h.md) · [`FVisual.cpp`](FVisual.cpp.md) · [`ModelPool.cpp`](ModelPool.cpp.md)
**Tier floor** — T1: it is a mesh record plus a draw.

## Purpose

Declares the surface implemented in [`FVisual.cpp`](FVisual.cpp.md). This is the model type most of a level is made of: every wall, floor, crate and prop that neither animates nor changes detail.

## Exported units

- **`StaticVisual`** — the model base with a mesh record folded into it, plus an optional second mesh: the *fast* geometry, a position-only version of the same shape used where only depth matters.
- **`load` / `render` / `copy` / `release`** — the base operations, filled in.

**Notes** — The class inherits both the model base and the mesh record rather than holding a mesh, which makes a static model *be* its geometry. That saves an indirection on the hottest draw path in the renderer and is why it is written that way; a rebuild is free to compose instead.
