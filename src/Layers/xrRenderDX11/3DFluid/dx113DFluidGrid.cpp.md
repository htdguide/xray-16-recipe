# src/Layers/xrRenderDX11/3DFluid/dx113DFluidGrid.cpp

> The geometry that drives every simulation pass: one quad per voxel slice for the interior, separate geometry for the boundary, because interior and boundary voxels obey different equations.

**Needs** — [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md) · [`xrRender/BufferUtils.h`](../../xrRender/BufferUtils.h.md) · [`../dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidGrid.h`](dx113DFluidGrid.h.md)
**Tier floor** — T1: it builds vertex buffers with an explicit byte layout and hands them to the device.

## Purpose

Every step of the fluid solver is "evaluate a function at every voxel of a three-dimensional field and write the result into another three-dimensional field". The device offers no primitive for that, so the subsystem manufactures one: a fixed set of vertex buffers that, when drawn, cause exactly one invocation per voxel with the voxel's grid coordinates available to it.

This file exists so that the manufacturing happens **once at startup**. The geometry depends only on the grid dimensions, which never change during a run, so all four buffers are built at initialization and then merely re-bound. The solver runs a dozen passes per volume per frame; re-deriving the geometry each time would dominate the cost.

The load-bearing idea is the **split between interior and boundary**. A voxel in the middle of the grid computes from its six neighbours. A voxel on the edge has no neighbour on one side, and what it should do there — mirror, clamp, zero the normal component — depends on which step is running and is the difference between fog that sits in its box and fog that leaks out of it. So the interior and the boundary are drawn by *different draw calls with different material passes*, and this file provides the geometry for each.

## State

```text
RECORD SliceVertex
  clip_position : (real, real, real)   # position in the projection's own space
  cell_coords   : (real, real, real)   # (x, y, slice index) in voxel units,
                                       # NOT normalized to 0..1
```

The second field is the whole trick: the rasterizer interpolates it across the quad, so each generated fragment receives its own integer-valued voxel coordinate, and the slice index rides along untouched to tell the pipeline which depth layer of the destination field to write.

```text
RECORD Grid
  dimensions   : (int, int, int)      # voxels
  max_dimension: int
  cols, rows   : int                  # the flat-atlas layout, see below
  four geometries, each a vertex buffer of SliceVertex:
    screen_slices   : 6 vertices per slice, all slices        # diagnostics
    interior_slices : 6 vertices per slice, slices 1..depth-2
    boundary_quads  : 6 vertices per slice, slices 0 and depth-1
    boundary_lines  : 2 vertices per line, 4 lines per slice, all slices
```

Invariant: `interior_slices` covers slices `1` through `depth-2` inclusive and each quad spans voxel coordinates `1` through `dimension-1`, so it never touches a face of the grid. `boundary_quads` covers exactly the two omitted slices at full extent, and `boundary_lines` covers the four edges of every slice. Together the three cover each voxel exactly once — that is the invariant a rebuild must preserve if it splits the work differently.

## `Initialize`

**Contract** — records the grid dimensions, computes the flat-atlas layout, and builds all four vertex buffers. Blocks; allocates device memory that lives until the subsystem is destroyed. Called once.

## `DrawSlices` — the interior pass

**Contract** — binds the interior geometry and issues one draw. Every interior voxel of the destination field receives exactly one fragment. This is the call that every simulation step ends with, and it is the reason the grid exists.

```text
FUNCTION interior_quad(z)
  # a quad inset by one voxel on every side of slice z
  corners in clip space:  voxel 1 .. voxel (width-1) horizontally
                          voxel 1 .. voxel (height-1) vertically
  cell coordinates match the corners in voxel units
  slice index = z, carried on every vertex
  emit as two triangles
```

**Invariants** — The mapping from voxel index to clip coordinate is `index * 2 / dimension - 1` horizontally and its negation vertically. **The vertical axis is negated**, which is the source of the downward-increasing vertical convention the rest of the subsystem compensates for (see [`dx113DFluidData.cpp`](dx113DFluidData.cpp.md), where authored emitter positions are mirrored). A rebuild is free to pick the other convention, but must pick it once and everywhere.

**Notes** — Two triangles per slice, six vertices, no index buffer. Indexing four vertices would halve the vertex count; with a few dozen slices the buffer is a few kilobytes either way and the simpler form won.

## `DrawBoundaryQuads` / `DrawBoundaryLines` — the boundary passes

**Contract** — `DrawBoundaryQuads` covers the first and last slices, which are whole faces of the grid. `DrawBoundaryLines` covers the four edges of every slice, which are the remaining four faces seen edge-on. A material pass that needs boundary handling runs after the interior pass and re-covers only these voxels.

**Notes** — The line geometry is submitted as a **triangle list** whose primitive count is the vertex count divided by three, although the vertices describe two-vertex line segments; the original call, preserved in the source as a comment, used a line list. Either the boundary passes are effectively dead in the shipped renderer, or they rasterize garbage that the material pass happens to tolerate. This is not recoverable from the source and a rebuild should submit lines as lines.

The boundary-line vertices carry a depth of one half rather than zero, and their cell coordinates are all zero rather than the voxel they cover. The material pass therefore cannot know which voxel it is shading from the interpolated coordinates — further evidence that this path is vestigial.

## `DrawSlicesToScreen` — the flat-atlas diagnostic

**Contract** — draws every slice of the volume side by side into a single two-dimensional image, laid out in a grid of rows and columns, so that the whole field can be inspected at once.

```text
FUNCTION atlas_layout(depth) -> (cols, rows)
  rows := floor(sqrt(depth))
  cols := rows
  WHILE rows * cols < depth
    cols := cols + 1
  # invariant: rows * cols >= depth, and the arrangement is as close to
  # square as an integer layout allows
```

**Notes** — The layout starts from a square and grows only the column count, so the atlas is never taller than it is wide. Nothing else in the subsystem uses this geometry; it is the debugging view that made the solver developable, and it is worth keeping for exactly that reason.
