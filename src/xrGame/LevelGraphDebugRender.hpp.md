# src/xrGame/LevelGraphDebugRender.hpp

> Declares the navigation-data overlay renderer, implemented in [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md).

**Needs** — [`Include/xrRender/DebugShader.h`](../Include/xrRender/DebugShader.h.md)
**Used by** — [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `LevelGraphDebugRender`, the instrument that draws the navigation data — the level's
walkable mesh, the cross-level graph, restrictor borders, cover values and the positions of
offline creatures. Substance is in
[`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md).

The whole declaration is compiled out of a shipped build. That is the shape decision: this
instrument is not a feature that can be enabled at run time, it is absent. A rebuild is free
to keep it always present behind a flag — nothing here costs anything when unused — and
probably should, because the data it reads is otherwise opaque.

Exported units:

- `LevelGraphDebugRender` — the renderer. Holds the two graphs for the duration of one draw,
  the untextured material the mesh quads use, the selected level slice with its cached
  extent, and a reused scratch list of nearby cover points.
- `Render` — draw every enabled overlay for this frame; takes both graphs.
- `SetupCurrentLevel` — which level's slice of the cross-level graph the miniature shows;
  -1 means all of them. Invalidates the cached extent.

Private steps, each a distinct overlay: the cross-level graph miniature and its coordinate
mapping, the walkable mesh near the camera, restrictor borders, per-vertex cover values at
two heights, per-object draw for creatures and smart covers, and a two-vertex manual probe.
