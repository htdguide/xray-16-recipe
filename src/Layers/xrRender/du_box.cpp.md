# src/Layers/xrRender/du_box.cpp

> The unit cube, in three forms: eight shared corners with a triangle index list, the same corners with a line index list, and thirty-six unshared vertices for a caller that cannot use indices.

**Needs** — [`du_box.h`](du_box.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`du_box.h`](du_box.h.md)
**Tier floor** — T1: a table of float triples and index words handed to the device as a vertex and index buffer.

## Purpose

One of five files holding the fixed unit primitives that the debug and editor drawing surface described in [`DrawUtils.h`](../../Include/xrRender/DrawUtils.h.md) draws. Each primitive is authored once as a constant table; a caller supplies a transform. The tessellation is *not* a parameter — every box the tools draw has the same eight corners, which is what makes a wireframe box recognizable as one.

## State

```text
CONSTANT du_box_vertices    : list<vec3>, 8 entries
CONSTANT du_box_faces       : list<int (16-bit)>, 12 triangles = 36 indices
CONSTANT du_box_lines       : list<int (16-bit)>, 12 edges = 24 indices
CONSTANT du_box_vertices2   : list<vec3>, 36 entries       # the same 12 triangles, unindexed
```

**Invariants**

- Every coordinate is ±0.5: the box is the **unit cube centred on the origin**, not the unit cube with a corner at the origin. A caller scales by the full extent, not the half extent. Every other primitive in this set shares the convention — see [`du_cylinder.cpp`](du_cylinder.cpp.md) and [`du_sphere_part.cpp`](du_sphere_part.cpp.md); [`du_cone.cpp`](du_cone.cpp.md) and [`du_sphere.cpp`](du_sphere.cpp.md) are the exceptions and say so.
- The corner order is not arbitrary and must be reproduced: indices 0..3 are the face at −Z walked counter-clockwise, indices 4..7 the face at +Z but **not** in the matching order — 4 and 5 are swapped relative to a naive mirror. The face and line tables are written against this exact ordering, so renumbering the corners silently inverts triangles.
- The triangle winding of `du_box_faces` is consistent across all twelve triangles, so the box can be drawn solid with back-face culling on. `du_box_lines` lists each of the twelve edges once; drawing the face list in line mode instead would draw the six face diagonals as well.
- `du_box_vertices2` is the same geometry expanded to thirty-six independent vertices. It exists for the draw path that has no index buffer to spare. Its triangle order differs from the indexed list's — it is not a mechanical expansion — so the two are independent tables that happen to describe the same cube.
