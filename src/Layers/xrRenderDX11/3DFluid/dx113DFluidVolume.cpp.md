# src/Layers/xrRenderDX11/3DFluid/dx113DFluidVolume.cpp

> The fluid volume as a scene object: how a fog volume enters the renderer's visibility and sorting machinery, and where its per-frame simulate-then-draw actually happens.

**Needs** — [`dx113DFluidVolume.h`](dx113DFluidVolume.h.md) · [`dx113DFluidData.h`](dx113DFluidData.h.md) · [`dx113DFluidManager.h`](dx113DFluidManager.h.md) · [`xrRender/FBasicVisual.h`](../../xrRender/FBasicVisual.h.md)
**Used by** — [`dx113DFluidVolume.h`](dx113DFluidVolume.h.md)
**Tier floor** — T2: it is a scene-graph node; its device work is entirely delegated.

## Purpose

Everything the renderer draws is a *visual* — a node with a bounding volume, a material and a draw method. A fluid volume is one of these, which is how it inherits visibility culling, sorting and the level's object lifecycle for free. This file is the adapter, and it is thin on purpose.

The one decision worth stating: **the simulation runs inside the draw call**, not in a separate update phase. A volume that is culled is therefore not simulated at all — its fields simply stop advancing and resume when it becomes visible again. That is a deliberate cost control and it is visible in play as fog that is already settled when you turn to look at it.

## State

```text
RECORD FluidVisual EXTENDS Visual
  data      : FluidVolume      # the fields, placement, obstacles, emitters
  debug_geometry : Geometry    # a unit box, kept for diagnostics
```

## `Load`

**Contract** — reads the volume's record and establishes it as a scene object: assigns a material whose only job is to place the volume correctly in the draw order, builds the debug geometry, marks the node's type, and derives the bounding box and sphere by transforming the **unit cube centred on the origin** by the volume's placement transform.

**Invariants** — The unit cube from −0.5 to +0.5 is the fluid's own space, and every part of the subsystem agrees on it: emitter positions, obstacle transforms and the ray-marching renderer all work in it. The placement transform is the only thing that maps it into the world.

**Notes** — The material is a stand-in named so that it does not begin with a digit (material names become script identifiers, which may not). It contributes no passes of its own; it exists so the sorting layer files the volume among the translucent objects, after opaque geometry and before the interface.

## `Render`

**Contract** — called by the renderer when the volume is visible. Fills the debug geometry with a unit box, sets the world transform, and then runs the simulation step and the volume draw.

```text
FUNCTION render(command_list)
  build a 24-vertex unit box into the shared dynamic vertex stream   # diagnostics only
  set the world transform to the volume's placement
  # the actual draw of that box is disabled; the vertices are built regardless
  FOR EACH obstacle: set the world transform to it  # likewise, draw disabled
  solver.update(this volume, fixed timestep)
  solver.render(this volume)
```

**Invariants** — The **timestep is a fixed constant of 2.0**, not the frame's elapsed time. The commented-out alternative derived it from the frame delta. Fixing it makes the simulation frame-rate dependent — the fog evolves faster on a faster machine — which is a real behavioural quirk of the shipped engine, but it also makes the solver unconditionally stable regardless of frame time, which a semi-Lagrangian scheme with a fixed iteration count needs. A rebuild that wants frame-rate independence must clamp the timestep and probably raise the pressure iteration count.

**Notes** — The debug box geometry is still built every frame although both of its draw calls are commented out. It costs two dozen vertices in the dynamic stream; it survives because it is the only way to see where a fog volume and its obstacles actually are.

## `Copy` / `Release`

**Contract** — delegate to the base visual. A fluid volume is never instanced, so there is nothing volume-specific to copy.
