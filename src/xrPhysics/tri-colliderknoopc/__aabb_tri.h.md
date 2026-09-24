# src/xrPhysics/tri-colliderknoopc/__aabb_tri.h

> Does this triangle touch this axis-aligned box? The rejection test that every triangle in the
> mesh collider passes through, twice, at two different strengths.

**Needs** — [`../../xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md)
**Used by** — [`PHSimpleCharacter.cpp`](../PHSimpleCharacter.cpp.md) · [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) · [`dTriColliderMath.h`](dTriColliderMath.h.md)
**Tier floor** — T1: a branch-ordered hot loop where the ordering *is* the optimisation.

## Purpose

The mesh collider fetches a few dozen to a few hundred candidate triangles per shape per step
from the static collision database, and most of them do not touch the shape. This file is how
they are thrown away. It is the most-executed code in the chapter, which is why it is written
as it is.

Two strengths of the same test are offered and the collider uses **both, on the same
triangle, at different moments**. That is the decision worth carrying across:

- a **cheap** form — three slab tests plus the triangle-plane test — used to skip a triangle
  outright;
- a **full** form — the same, plus the nine edge-cross axes — used only once a triangle has
  already been found interesting, to decide whether it really overlaps.

The cheap form over-reports. Running only the full form everywhere costs more than the
over-reporting does, and running only the cheap form produces contacts against triangles the
shape does not touch.

## Stateless.

## `aabb_tri_aabb` — the cheap form

**Contract** — reject if the triangle and the box are separated along any of the three world
axes, or if the box does not straddle the triangle's plane. Otherwise report overlap.

```text
FUNCTION cheap_overlap(centre, half_extents, vertices) -> bool
  FOR EACH world axis a                  # done one axis at a time, x then y then z
    translate the three vertices' a-component into the box's frame
    IF all three are below -half_extents[a] THEN RETURN false
    IF all three are above +half_extents[a] THEN RETURN false
  # now the plane test
  normal := cross(v1 - v0, v2 - v1)
  d      := -dot(normal, v0)
  RETURN |d| < |half_extents ⋅ |normal||     # the box straddles the plane
```

**Invariants** — the vertices are translated component-by-component, and the y and z
components are computed *only after* the x test passes. That interleaving is the point: a
triangle rejected on x never has its y or z components loaded. In a rebuild this reads as
"early-out per axis before computing the next axis", not as three separate loops.

**Notes** — the axis test is written as "are all three on the far side", not as "compute the
triangle's min and max and compare". Both are correct; the first short-circuits on the first
vertex that is inside, and for the common case — a triangle that overlaps — that is cheaper.

The plane-straddle test has two implementations here, and comparing them is instructive. The
obvious one builds the box's two extreme corners along the normal and tests both. The one
actually used observes that the box is symmetric about its centre, so the whole thing reduces
to comparing the plane's offset against the box's half-extent projected onto the normal — one
absolute-value sum instead of six sign branches. In a checked build the two are run against
each other and any disagreement is logged, which is a nice pattern to keep: the slow, obviously
correct version survives as the fast one's oracle.

## `__aabb_tri` — the full form

**Contract** — the cheap form, then the nine cross-product axes: each of the triangle's three
edges crossed with each of the three world axes.

```text
FUNCTION full_overlap(centre, half_extents, vertices) -> bool
  IF NOT cheap_overlap(...) THEN RETURN false
  FOR EACH edge e OF the triangle
    FOR EACH world axis a
      axis := cross(e, a)
      project the triangle onto axis      # two vertices suffice; the third is on
                                          # the edge and projects between them
      project the box onto axis           # a sum of two terms; the third is zero
      IF the intervals do not overlap THEN RETURN false
  RETURN true
```

**Invariants** — this is the complete separating-axis set for a box against a triangle:
three box face normals, one triangle plane normal, nine edge crosses. Dropping any of the nine
lets a triangle pass diagonally through a box corner undetected.

**Notes** — three specialisations make the nine tests cheap, and all three are worth keeping
because they are algebraic, not micro-optimisations:

1. **Only two of the three vertices are projected per axis.** The axis is perpendicular to one
   edge, so that edge's two endpoints project to the same value and the third vertex is the
   only one that can differ. Which two depends on which edge, which is why the tests are not
   written as one loop.
2. **Only two of the three box half-extents contribute.** The axis is perpendicular to a world
   axis, so that component of the box projects to zero.
3. **The absolute values of the edge components are computed once** and reused across the
   three axes that share an edge, and the edges themselves are built lazily so that an early
   rejection never computes the last one.

The original prefixes the file name with underscores and leaves the whole thing as macros; both
are artefacts of it having been lifted from a triangle-mesh library of the era. What survives
is the two-strength structure and the axis set.

## `Point`

**Contract** — a three-component vector with the usual arithmetic, a dot product (written as a
bitwise-or operator), a cross product (written as exponentiation), array access, and
conversion to and from a bare three-float array.

**Notes** — this exists because the file was lifted with its own vector type rather than
adapted to the engine's, and it converts to a raw float triple so the two can be aliased. A
rebuild uses its own vector and deletes this. The only thing worth noticing is that it is
exactly three floats with no padding, so a triangle's vertices can be read straight out of the
collision database's vertex array with no copy — which the collider does, in its hot loop, and
which is one reason this chapter sits at T1.
