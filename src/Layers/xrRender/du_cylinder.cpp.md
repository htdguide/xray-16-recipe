# src/Layers/xrRender/du_cylinder.cpp

> The unit cylinder: a twelve-sided tube of radius 0.5 running from z = −0.5 to z = +0.5, capped at both ends.

**Needs** — [`du_cylinder.h`](du_cylinder.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`du_cylinder.h`](du_cylinder.h.md)
**Tier floor** — T1: a table of float triples and index words handed to the device as a vertex and index buffer.

## Purpose

One of the five fixed unit primitives described in [`du_box.cpp`](du_box.cpp.md); the tessellation is authored into the table and is not a caller's choice.

## State

```text
CONSTANT du_cylinder_vertices : list<vec3>, 26 entries
CONSTANT du_cylinder_faces    : list<int (16-bit)>, 48 triangles = 144 indices
CONSTANT du_cylinder_lines    : list<int (16-bit)>, 30 edges = 60 indices
```

**Invariants**

- Centred on the origin and **one unit tall**, radius **0.5** — the same convention as the box. A caller scales by the full diameter and the full height.
- The twelve-sided ring steps by 30°. The vertices are **interleaved by end, not grouped**: even indices lie on the +Z cap and odd indices on the −Z cap, alternating, starting from +X. Indices 24 and 25 are the two cap centres, +Z then −Z. The face and line tables are written against this interleaving; grouping the ring by end instead breaks both.
- The face list is twenty-four side triangles (two per quad of the tube) followed by twelve triangles for each cap, wound so the closed solid faces outwards throughout.
- The line list is the same economy as the cone: only **six** of the twelve axis-parallel edges are drawn — every other one — plus all twelve segments of each cap ring. The skipped six remain in the source as comments, and the header records the full count of 36 beside the used count of 30.
