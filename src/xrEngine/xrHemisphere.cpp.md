# src/xrEngine/xrHemisphere.cpp

> Three baked, evenly-distributed hemisphere tessellations — used as the sky dome's mesh and as a fixed set of light directions.

**Needs** — [`xrHemisphere.h`](xrHemisphere.h.md)
**Used by** — [`xrHemisphere.h`](xrHemisphere.h.md)
**Tier floor** — T2: constant tables and a loop. The tables themselves are data, not code, and a rebuild should carry them as data.

## Purpose

Two unrelated jobs want the same thing: a set of directions spread evenly over a
hemisphere. The sky needs a dome to draw the cloud layer on; a lighting pass needs a fixed
set of sample directions to integrate incoming light over. Both are served by three
precomputed tessellations at increasing density.

The vertex positions are *baked constants*, not generated. That is the decision this file
exists to record: the distributions are fixed, identical on every machine and every run, and
reproducing them exactly matters more than being able to derive them.

## State

```text
Three tessellations, selected by a quality level 1..3:

  quality 1   26 vertices,  40 triangles     # icosahedral subdivision, one level
  quality 2   91 vertices, 160 triangles     # icosahedral subdivision, two levels
  quality 3  196 vertices,  (no triangles)   # directions only
```

Invariants:

- Quality 1 and 2 are unit-length direction vectors with a triangle list; quality 3 is
  **not** — its vectors have magnitude about one half, and it has no triangle list at all.
  The consumer normalises, which is why the discrepancy is invisible; a rebuilder copying
  the tables must not assume unit length.
- The upper (positive-height) hemisphere is what is stored, with the equator included as a
  full ring so the dome closes cleanly against a horizon.
- The two lower qualities descend from an icosahedron, which is why their vertex counts are
  what they are rather than round numbers.

## `xrHemisphereVertices`

**Contract** — hands back the direction table for a quality level and returns its length.
The table is shared, immutable and outlives the caller. An unknown quality level is a
programming error, not a runtime condition: the function does not return a default.

## `xrHemisphereIndices`

**Contract** — the same for the triangle list, returning the number of *indices* (three per
triangle). Only qualities 1 and 2 have one; asking for quality 3 is a programming error.

**Notes** — the missing quality-3 triangle list is commented out in the source with no
explanation. Either the table was never produced or it was dropped; **this is not
recoverable**. The effect is that the highest quality is usable for light sampling and not
for drawing.

## `xrHemisphereBuild`

**Contract** — calls a supplied function once per direction in the chosen tessellation,
passing the *negated, normalised* direction and an equal share of a total energy. Allocates
nothing; the callback carries the caller's own context.

```text
FUNCTION hemisphere_build(quality, total_energy, visit, context)
  directions = table for quality
  share = total_energy / count(directions)
  FOR EACH d IN directions
    v = normalise(-d)          # negate: the table points outward, lights point inward
    visit(v, share, context)
```

**Invariants** — the energy is split *equally* across samples. This is only correct because
the tessellation is near-uniform on the sphere; a distribution with varying cell area would
need per-sample weights. The equal split is the reason the baked tables had to be
icosahedral rather than, say, a latitude/longitude grid, which clusters at the pole.

**Notes** — the negation converts "a point on the dome" into "the direction light arrives
from at the origin". The normalisation is unconditional and is what rescues the quality-3
table's non-unit vectors.

In the shipped engine only the sky dome uses these tables, and only at quality 2. The
lighting use the iterator was written for lives in the offline level compiler, which is not
part of this repository — so the callback form survives here as an unused generality.
