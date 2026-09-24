# src/Layers/xrRender/du_sphere_part.cpp

> A closed wedge of a sphere — a cap of radius 0.5 around +Z, joined to the centre — used to draw a cone of vision or an audible arc as a solid the eye reads as a piece of a ball.

**Needs** — [`du_sphere_part.h`](du_sphere_part.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`du_sphere_part.h`](du_sphere_part.h.md)
**Tier floor** — T1: a table of float triples and index words handed to the device as a vertex and index buffer.

## Purpose

One of the five fixed unit primitives described in [`du_box.cpp`](du_box.cpp.md), and the only one whose authored shape encodes an angle rather than a unit extent.

## State

```text
CONSTANT du_sphere_part_vertices : list<vec3>, 82 entries
CONSTANT du_sphere_part_faces    : list<int (16-bit)>, 160 triangles = 480 indices
CONSTANT du_sphere_part_lines    : list<int (16-bit)>, 176 segments = 352 indices
```

**Invariants**

- **Radius 0.5, centred on the origin, opening along +Z.** The first eighty-one vertices lie on the sphere of radius 0.5; the last, index **81, is the origin itself**. That final vertex is what closes the wedge: the outer triangles tile the cap and a second family of triangles fans the cap's boundary back to the centre, so the shape is a solid, not an open shell. A rebuild that drops the centre vertex gets a cap that is invisible from behind.
- The cap subtends roughly a right angle about +Z — its boundary vertices sit at about 45° from the axis. The extent is authored into the table and is not a parameter; a caller that wants a narrower cone scales X and Y down and Z up, which squashes the cap rather than re-tessellating it.
- The line list is much denser than the face list is long (176 segments over 82 vertices) because it draws *every* edge of the tessellation plus every spoke to the centre. Unlike the cone and the cylinder, nothing is skipped here: the wedge is drawn small and needs the density to read as curved.
- The vertex coordinates in the source carry no floating-point suffix and are therefore written as double-precision literals narrowed on assignment. That is a language artifact; the values are single-precision quantities and the narrowing is exact for all of them.

**Notes** — This is the only primitive in the set whose shape encodes an *angle*. Where the cone's opening is entirely the caller's (it scales radius against length), this one's is baked in, so a caller drawing a 30° field of view and a caller drawing a 120° one get visibly different curvature on the cap. That is a limitation of the authored table, not a decision worth preserving if a rebuild can tessellate on demand — but the shipped tools' output changes if it does.
