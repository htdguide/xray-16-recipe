# src/Layers/xrRender/du_cone.cpp

> The unit cone: an apex at the origin and a sixteen-sided cap one unit along +Z.

**Needs** — [`du_cone.h`](du_cone.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`du_cone.h`](du_cone.h.md)
**Tier floor** — T1: a table of float triples and index words handed to the device as a vertex and index buffer.

## Purpose

One of the five fixed unit primitives described in [`du_box.cpp`](du_box.cpp.md); the tessellation is authored into the table and is not a caller's choice.

## State

```text
CONSTANT du_cone_vertices : list<vec3>, 18 entries
CONSTANT du_cone_faces    : list<int (16-bit)>, 32 triangles = 96 indices
CONSTANT du_cone_lines    : list<int (16-bit)>, 24 edges = 48 indices
```

**Invariants**

- The layout is fixed: index 0 is the **apex at the origin**; indices 1..16 are the rim, sixteen points on a circle of radius **0.5** in the plane z = 1, starting at +X and turning towards +Y; index 17 is the rim's **centre**, at (0, 0, 1).
- The cone therefore points **backwards** along its own axis: it opens towards +Z and its tip is at the origin, so a caller places the apex — the light's position, the sound's origin — and scales the length along Z and the radius in X and Y independently. This is the opposite convention from the box: nothing is centred.
- Radius 0.5 with height 1 means a caller scaling uniformly gets a 90° full angle. A spot light's cone is drawn by scaling X and Y by `tan(half_angle) * range * 2` and Z by `range`.
- The sixteen rim points give the cone a recognizable silhouette at the sizes the tools draw it. The number is fixed by the tables and is not a parameter.
- The face list is the sixteen side triangles (apex to consecutive rim pairs) followed by the sixteen cap triangles (centre to consecutive rim pairs), with the cap wound opposite to the sides so the solid is closed and consistently facing outwards.
- The line list draws only **eight** spokes — every other rim point — plus all sixteen rim segments. Drawing all sixteen spokes turns the wireframe into a solid-looking fan at typical screen sizes; the alternating pattern is deliberate and the skipped spokes are still present in the source as comments.
