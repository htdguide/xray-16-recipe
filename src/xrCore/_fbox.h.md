# src/xrCore/_fbox.h

> The axis-aligned bounding box: two corner points, and the accumulate / test / transform vocabulary that every spatial structure in the engine is built out of.

**Needs** — [`_vector3d.h`](_vector3d.h.md) · [`_matrix.h`](_matrix.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md)

**Used by** — [`LevelStructure.hpp`](../Common/LevelStructure.hpp.md) · [`ISpatial.cpp`](../xrCDB/ISpatial.cpp.md) · [`ISpatial_q_box.cpp`](../xrCDB/ISpatial_q_box.cpp.md) · [`ISpatial_q_frustum.cpp`](../xrCDB/ISpatial_q_frustum.cpp.md) · [`ISpatial_q_ray.cpp`](../xrCDB/ISpatial_q_ray.cpp.md) · [`xrCDB_Collector.cpp`](../xrCDB/xrCDB_Collector.cpp.md) · [`xrCDB_ray.cpp`](../xrCDB/xrCDB_ray.cpp.md) · [`dump_string.cpp`](dump_string.cpp.md) · [`vector.h`](vector.h.md) · [`vis_common.h`](../xrEngine/vis_common.h.md) · [`ik_collide_data.h`](../xrGame/ik_collide_data.h.md) · [`particle_core.cpp`](../xrParticles/particle_core.cpp.md)

**Tier floor** — T1: six consecutive reals with no header and no padding, written into level files and collision trees as a memory image, and passed by value through visibility loops.

## Purpose

Every renderable, every collision node, every sector, every spatial-hash cell and every
level's extent is one of these. The type is small and its operations are short, but the
*set* of operations is the thing to take away: a rebuild that implements only "contains" and
"intersects" will find itself reimplementing the other twenty in a dozen places.

The two ideas that make the type work are on this page and nowhere else: the **invalid** box
that is the identity for accumulation, and the **transform** that keeps the result
axis-aligned without transforming eight corners.

## State

```text
RECORD Box3
  minimum : (real, real, real)    # stored first
  maximum : (real, real, real)
```

Exactly six reals in that order — minimum's x, y, z then maximum's x, y, z — and the type
exposes a pointer to that run because several loaders read and write it whole.

**Invariants**

- A box is **valid** when every maximum component is greater than or equal to the matching
  minimum. A box that fails this is not "empty"; it is either the accumulation identity
  described next, or a bug.
- The **invalid** box sets every minimum component to the largest representable real and
  every maximum to the smallest. It is deliberately the *inverse* of a legal box, and it is
  the identity element for the accumulate operation: extending it by any point yields the
  degenerate box at that point. Accumulation loops therefore start by invalidating and never
  need a first-iteration special case. This is the single most reused idea in the type and
  a rebuild must keep it.
- The **identity** box is the unit cube centred on the origin, spanning -0.5 to +0.5 on each
  axis. It is not the zero box and it is not the invalid box; three different "empty-ish"
  values exist and mean three different things. The unit cube is the one the transform
  helpers assume, because a box's transform is defined as "the transform that maps the unit
  cube onto this box".

## Construction and reset

**Contract** — Set from two corners; from six components; from another box; from a centre
and a half-extent; to all zeros; to the unit cube; to the accumulation identity. All
allocation-free, all returning the box so calls chain.

**Notes** — The centre-and-half-extent form takes the *half* extent, not the full size,
which is the opposite of the size accessor's convention. Nothing enforces it and a rebuild
should name the two differently.

## Growth and shift

**Contract** — Four families, each in a uniform-scalar and a per-axis form:

- **grow** — push the corners apart, making the box larger.
- **shrink** — pull them together. A shrink larger than the box's half-size inverts it and
  is not checked.
- **add / offset** — translate both corners. Two names for one operation; the duplication is
  an artifact.
- **scale** — grow by a fraction of the box's own size. The argument is a fraction, not a
  factor: passing one tenth makes the box 110% of its size, and passing minus one tenth makes
  it 90%. It is applied as a *grow* by a per-axis amount, so a non-cubic box grows more along
  its long axis than its short one.

## `modify` — accumulate a point

**Contract** — Extend the box to contain a point, by taking the componentwise minimum and
maximum. This is the accumulation primitive; almost every bounding box in the engine is built
by invalidating and then modifying once per vertex.

## `merge` — accumulate a box

**Contract** — Extend to contain another box, or build a box containing two others. The
two-argument form invalidates first, so it is a construction rather than an accumulation —
a distinction worth naming in a rebuild, because the one-argument form does *not* invalidate
and the two are easy to confuse.

## `contains`, `intersect`, `similar`

**Contract** — Three predicates, all pure and inclusive at the boundary:

- **contains** a point, or another box (both its corners). Touching the face counts as inside.
- **intersect** — overlap against another box, by six separating-axis comparisons. Touching
  faces count as overlapping, because the comparisons are strict; a rebuild that flips them
  to non-strict changes which neighbouring spatial cells report as adjacent.
- **similar** — corners equal within the loose epsilon.

## `xform` — the transformed bound

**Contract** — Produces the axis-aligned box containing the transform of another box.
Allocation-free. The in-place form copies the source out of the way first, because the
algorithm reads the source while writing the destination.

```text
FUNCTION transformed_bound(box, transform) -> Box3
  # The three edge vectors of the box, transformed. An edge that runs along
  # one axis only picks up one row of the transform, so this is three scales
  # rather than three full transforms.
  edge_x = transform.row1 * (box.maximum.x - box.minimum.x)
  edge_y = transform.row2 * (box.maximum.y - box.minimum.y)
  edge_z = transform.row3 * (box.maximum.z - box.minimum.z)

  # Start with the transformed minimum corner as a degenerate box.
  result.minimum = transform_point(box.minimum)
  result.maximum = result.minimum

  # Each transformed edge extends the box in one direction per component:
  # a negative component pushes the minimum out, a positive one pushes the
  # maximum. Summing all three components of all three edges reaches exactly
  # the two extreme corners of the transformed box.
  FOR EACH edge IN (edge_x, edge_y, edge_z)
    FOR EACH component c IN (x, y, z)
      IF edge.c is negative THEN result.minimum.c += edge.c
      ELSE                       result.maximum.c += edge.c
  RETURN result
```

**Invariants** — This is exact, not conservative: the result is the tightest axis-aligned
box containing the transformed box, and it costs one point transform plus three scaled row
reads instead of eight point transforms. It works for any affine transform, including ones
with scale and shear, and is wrong for a transform with a projection column.

**Notes** — The sign test is written as an inspection of the real's sign bit rather than a
comparison. That is an optimization from an era when a floating-point compare stalled; the
decision it encodes is simply "which of the two corners does this contribution extend".
Negative zero tests as negative under a sign-bit inspection and as non-negative under a
comparison — it extends the minimum by zero either way, so the two are equivalent here.

## `modify(source, transform)` — the corner-by-corner alternative

**Contract** — Extends this box by all eight transformed corners of another. Slower than
`xform` and *accumulates* rather than replaces, which is the reason both exist: a bound over
several transformed boxes is one invalidate and several of these.

## `get_xform` and the measurement accessors

**Contract** — A family of read-only shapes over the same two corners:

| Operation | Result |
|---|---|
| size | maximum minus minimum — the full extent per axis |
| radius, as a vector | half the size |
| radius, as a scalar | the length of that half-size — the circumradius, not the inradius |
| volume | the product of the three sizes |
| centre | the midpoint |
| centre and dimensions together | both, sharing the subtraction |
| sphere | the centre and the distance from it to the maximum corner — the circumscribed sphere |
| transform | the affine transform mapping the unit cube onto this box: translate to the centre, then scale by the half-extent |

**Notes** — `get_xform` builds the transform by setting a translation and then *setting* a
scale, which overwrites the translation's linear part rather than composing with it. The
result is a transform whose linear part is the scale and whose translation is the centre —
which happens to be exactly what was wanted, because a diagonal scale and a translation do
not interact. A rebuild that composes properly gets the same answer; a rebuild that copies
the shape of this code into a case with a rotation does not.

## `getpoint` and `getpoints` — the eight corners

**Contract** — One corner by index, or all eight into a caller's array. The ordering is
fixed and is relied upon by the debug renderer's wireframe and by the corner-by-corner
transform:

```text
0: (min.x, min.y, min.z)    4: (min.x, max.y, min.z)
1: (min.x, min.y, max.z)    5: (min.x, max.y, max.z)
2: (max.x, min.y, max.z)    6: (max.x, max.y, max.z)
3: (max.x, min.y, min.z)    7: (max.x, max.y, min.z)
```

**Invariants** — The first four are the bottom face traversed as a ring, and the last four
are the top face traversed the same way, so consecutive indices within a group are always
edge-adjacent and index `i` is directly below index `i+4`. That adjacency is what lets a
wireframe be drawn as two four-line loops plus four verticals. An out-of-range index yields
the origin rather than failing.

## `Pick` — does the ray reach the box

**Contract** — A boolean line test against the box. Takes an origin and a direction, returns
whether the *infinite* line passes through the box. Pure, allocation-free.

```text
FUNCTION line_hits_box(origin, direction) -> bool
  # Work relative to the origin, so each face plane is at a known offset.
  low  = minimum - origin
  high = maximum - origin

  # For each axis the direction is not parallel to, walk to each of that
  # axis's two face planes and test whether the other two coordinates land
  # inside the face. Six planes, and any one hit is enough.
  FOR EACH axis a WITH direction.a not near zero
    FOR EACH plane IN (low.a, high.a)
      t = plane / direction.a
      IF the other two coordinates of t*direction lie between low and high
        RETURN true
  RETURN false
```

**Notes** — No distance is returned and the sign of the parameter is never examined, so this
reports hits *behind* the origin as hits. It answers "is the box on this line", not "does
this ray reach the box". Several callers want the second and get the first; the narrowing
form below is what they should use.

## `Pick2` — the ray hit with its point

**Contract** — The candidate-plane ray test: returns the three-way classification shared
with the sphere and the cylinder, and writes the entry point when there is one. Pure and
allocation-free.

```text
FUNCTION ray_hits_box(origin, direction) -> (classification, point)
  inside = true
  FOR EACH axis a
    IF origin.a < minimum.a THEN
      point.a = minimum.a ; inside = false
      IF direction.a is non-zero THEN t[a] = (minimum.a - origin.a) / direction.a
    ELSE IF origin.a > maximum.a THEN
      point.a = maximum.a ; inside = false
      IF direction.a is non-zero THEN t[a] = (maximum.a - origin.a) / direction.a
    # An origin already within the slab leaves t[a] at its initial -1,
    # which is what makes it lose the "largest t" comparison below.

  IF inside THEN RETURN (origin_inside, origin)

  # The entry point is on whichever candidate plane is crossed LAST.
  winner = the axis with the largest t
  IF t[winner] is negative THEN RETURN (no_hit)      # the box is behind us

  # Confirm the crossing actually lands inside the face, not outside its edge.
  FOR EACH axis a OTHER THAN winner
    point.a = origin.a + t[winner] * direction.a
    IF point.a outside [minimum.a, maximum.a] THEN RETURN (no_hit)

  RETURN (origin_outside, point)
```

**Invariants**

- Only the coordinates of axes the origin is *outside* on are written into the point. The
  winning axis's coordinate was written during the candidate-plane scan; the other two are
  written during the confirmation. An axis whose origin is inside the slab and which is not
  the winner never gets written — so the caller's point must be initialized or the routine
  must be understood as writing only a partial point. The original leaves it uninitialized,
  which is a real hazard rather than a subtlety.
- The parameters start at -1 so that an axis contributing no constraint always loses the
  "largest" comparison. A rebuild using an explicit "no constraint" marker is clearer and
  equivalent.

**Notes** — Two tests here inspect the sign bit of a real rather than comparing: "is the
direction component non-zero" is written as "are any bits set", and "is the winning parameter
negative" as "is the sign bit set". The first is subtly different from a comparison — it
treats negative zero as non-zero and so divides by it, producing an infinite parameter that
then loses every comparison, which is harmless but arrives there by accident. A rebuild
compares and is correct for the right reason.
