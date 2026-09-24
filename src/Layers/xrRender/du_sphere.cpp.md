# src/Layers/xrRender/du_sphere.cpp

> The unit sphere: a twice-subdivided icosahedron for the solid form, and three great circles for the wireframe — two different meshes, because a subdivided icosahedron makes a terrible wire sphere.

**Needs** — [`du_sphere.h`](du_sphere.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`du_sphere.h`](du_sphere.h.md)
**Tier floor** — T1: a table of float triples and index words handed to the device as a vertex and index buffer.

## Purpose

One of the five fixed unit primitives described in [`du_box.cpp`](du_box.cpp.md). Unusually it holds **two** meshes: a solid one and an entirely separate wire one, because the wireframe of the solid mesh is unreadable.

## State

```text
CONSTANT du_sphere_vertices  : list<vec3>, 92 entries   # solid form
CONSTANT du_sphere_faces     : list<int (16-bit)>, 180 triangles = 540 indices
CONSTANT du_sphere_verticesl : list<vec3>, 60 entries   # wire form
CONSTANT du_sphere_lines     : list<int (16-bit)>, 60 segments = 120 indices
```

**Invariants**

- **Radius 1, centred on the origin** — unlike the box, the cylinder and the sphere-part, which are all radius 0.5 or half-extent 0.5. A caller of the sphere scales by the radius; a caller of the box scales by the diameter. Mixing the two conventions is the easy mistake here and the tables are the only documentation of which is which.
- The solid form is an icosahedron subdivided twice and renormalized: the first twelve vertices are the icosahedron's own, and the remaining eighty are the edge and face midpoints pushed out onto the sphere. 180 triangles is 20 × 9, which is what two subdivision steps of an icosahedron give if the second step splits each triangle into four and the first into… it does not decompose evenly, and the table is authored data rather than the output of a rule a rebuild can re-derive. **Copy the table.**
- The wire form is a *separate* vertex set: three closed rings of twenty segments each, in the XY, XZ and YZ planes, at radius 1. Each ring's first and last vertices are distinct entries that happen to coincide with neighbouring rings' — the rings are not stitched and share nothing. Drawing the solid form's edges instead would produce a dense ball of 270 lines that reads as a filled circle.
- Twenty segments per ring is the fixed tessellation of every wire sphere the engine draws — a creature's audible radius, a light's reach, a collision sphere. They are recognizable because they are all the same.

**Notes** — The two forms cannot be unified without changing how the tools look. A rebuild that wants one mesh should keep the wire rings and drop the solid form, not the other way round: the solid sphere is drawn rarely, the wire sphere constantly.
