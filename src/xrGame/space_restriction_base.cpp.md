# src/xrGame/space_restriction_base.cpp

> Decides what it means for a navigation vertex to be inside a volume — by testing the vertex's four corners and its centre — and fixes the spatial sort order that every border lookup depends on.

**Needs** — [`space_restriction_base.h`](space_restriction_base.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`space_restriction_base.h`](space_restriction_base.h.md)
**Tier floor** — T2: five geometric tests per call, called across whole regions of the navigation mesh

## Purpose

A navigation vertex is not a point; it is a square cell of the navigation mesh with a
sloped surface. A volume can cover all of it, part of it, or none. This file is where that
three-way distinction is defined, and the definition is consulted by everything from
pathfinding to border construction, so it is one of the most-executed pieces of the
chapter.

## State

`Stateless.` — operates on the border list held by the abstract base.

## `inside` (vertex, partially)

**Contract** — reports whether a navigation vertex is inside the volume. The
`partially` flag selects the meaning: with it, *any* of five sample points being inside
suffices; without it, *all five* must be. Delegates to the sphere test five times.
Hard-fails if the vertex identifier is not valid.

```text
FUNCTION inside(vertex, partially, radius = tiny) -> bool
  half = navigation cell size / 2 - tiny
  centre = vertex_position(vertex)
  samples = [ (centre.x + half, centre.z + half),
              (centre.x + half, centre.z - half),
              (centre.x - half, centre.z + half),
              (centre.x - half, centre.z - half) ]
  # each corner's height comes from the vertex's own sloped plane, not from the centre
  points = [ (s.x, plane_height(vertex, s.x, s.z), s.z) FOR s IN samples ] + [ centre ]

  IF partially
    RETURN ANY p IN points SATISFIES inside(sphere(p, radius))
  RETURN ALL p IN points SATISFY inside(sphere(p, radius))
```

**Invariants** — the four corners are pulled inward by a small epsilon so that two
adjacent cells never both claim the shared edge. Without that, a volume whose boundary
runs exactly along a cell edge would be judged to contain both neighbours, and the border
would be two cells thick in places and one in others.

**Notes** — the corner heights are evaluated on the vertex's own *plane*, not at the
centre's height. Navigation cells are sloped, and on a staircase or a ramp a volume that
covers the low corner of a cell but not the high one must be seen to do so; using a single
height would make every sloped cell's answer depend on where in the cell the volume
happens to sit.

The default radius is a near-zero epsilon rather than zero, because a genuinely
zero-radius sphere degenerates in the volume tests. The overload taking a radius exists so
that a caller can ask on behalf of a body with a real width — that is how a wide creature
is kept from a passage a narrow one may take.

Five samples is a deliberate balance: four corners alone would let a small volume sitting
in the middle of a cell escape the partial test entirely, and a full grid would cost more
than the answer is worth on a path search.

## `process_borders`

**Contract** — puts the border list into its canonical form: deduplicated, then sorted by
the vertex's packed horizontal grid coordinate. Called once at the end of every border
build. Allocates nothing beyond the sort.

```text
FUNCTION process_borders()
  SORT border BY vertex id ; REMOVE duplicates
  SORT border BY packed_xz(vertex)     # the final, load-bearing order
```

**Invariants** — after this, the border is ordered by horizontal grid position. **That
ordering is a contract, not a tidiness measure**: the "is this position on the border"
query in [`space_restriction_bridge.cpp`](space_restriction_bridge.cpp.md) binary-searches
this list by exactly that key, and the intersection test in
[`space_restriction.cpp`](space_restriction.cpp.md) runs a sorted-set intersection over
it. A rebuild that leaves the border in any other order silently breaks both.

Deduplication happens under the *identifier* order and the final sort is under the
*position* order, so the final list may contain several vertices sharing one horizontal
cell — vertices stacked vertically, as under a bridge or on a multi-storey building. Both
consumers handle that by scanning the run of equal keys, which is why deduplication cannot
be folded into the second sort.

## `correct` (checked builds)

**Contract** — reports the result of the connectivity self-test that a concrete
restriction runs after building its border: whether flooding the navigation mesh inward
from inside the volume, with the border stamped as a barrier, reaches exactly the set of
vertices the volume was found to contain. A restriction that fails this has a leaky border
— a gap a creature could walk through — and the failure is reported by name at load time.
Debug-only; a rebuild should keep it, because it is the only automatic check that authored
restrictor geometry is usable.
