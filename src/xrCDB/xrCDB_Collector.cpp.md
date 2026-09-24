# src/xrCDB/xrCDB_Collector.cpp

> Accumulates a triangle soup from individually submitted faces, welding
> coincident vertices so the resulting index arrays are compact.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: array building and a spatial hash; the only reason to go lower is the
size of the arrays involved.

## Purpose

Producers of collision geometry — the level compiler, the editor, anything synthesizing a
collision proxy — emit triangles one at a time, in arbitrary order, with vertices repeated
wherever faces meet. The tree wants the opposite: two flat arrays, vertices deduplicated so
the index array is dense. This file is that adapter.

It holds two collectors that differ only in how they find an existing vertex: one scans,
one hashes. That is the whole distinction, and it is a real one — the scanning version is
quadratic and unusable on a level.

## State

```text
RECORD Collector
  verts : list<vec3>
  faces : list<Triangle>

RECORD CollectorPacked                 # same soup, plus an index for welding
  verts       : list<vec3>
  faces       : list<Triangle>
  face_flags  : list<int (32-bit)>     # one per face; see the note
  grid_min    : vec3                   # the bounding box the grid spans
  grid_scale  : vec3
  grid_eps    : vec3                   # per-axis weld tolerance, capped
  grid        : list<int>[25][17][25]  # cell -> indices of vertices in it

# Invariants (both)
#   every faces[i].v[k] is a valid index into verts
#   count(face_flags) == count(faces)         -- packed collector only
#   a vertex appears in verts at most once, up to the weld tolerance
```

The grid is `24 x 16 x 24` cells plus one, spanning the bounding box the collector is told
about at construction. The shape is not arbitrary: a level is wide and shallow, so the
vertical axis gets fewer cells than the horizontal ones. The `+1` is the overflow row that
catches a vertex landing exactly on the far boundary, which is cheaper than clamping the
common case. These three numbers are a tuning choice with no derivation available in the
source; any rebuild should treat them as *the grid resolution should reflect the level's
aspect ratio* and pick its own.

## `Collector.add_face`

**Contract** — appends a triangle with three fresh vertices and no welding at all: three
new entries in the vertex array every time. Takes either a material id and a sector id, or
the whole payload word directly. Allocates amortized.

**Notes** — two entry points exist, one taking the fields and one taking the packed word,
because a producer that is copying triangles from an existing soup already has the word and
unpacking and repacking it would lose the two cached material bits. Same distinction on
every other entry point here.

## `Collector.add_face_welded`

**Contract** — appends a triangle, reusing an existing vertex when one is within a caller-
supplied tolerance. Finds that vertex by scanning the whole vertex array, so a soup of `n`
vertices costs `O(n²)` to build. Intended for soups of a few hundred faces — a collision
proxy for one model — and nothing larger.

## `CollectorPacked.add_face`

**Contract** — the same, but welding is constant-time.

```text
FUNCTION weld(p) -> int
  cell <- the grid cell containing p
  FOR EACH i IN grid[cell]
    IF verts[i] is within tolerance of p: RETURN i

  index <- append p to verts
  add index to grid[cell]
  # also register it in every neighbouring cell it is within tolerance of
  cell_hi <- the grid cell containing (p + grid_eps)
  FOR EACH cell' IN the up-to-8 cells spanned by cell and cell_hi
    add index to grid[cell']
  RETURN index
```

**The multi-cell registration is the correctness of the whole scheme.** A vertex sitting
just inside a cell boundary must be findable from the neighbouring cell, or two vertices a
hair apart across a boundary weld into two distinct entries and the surface they share
develops a crack a ray can pass through. Registering the vertex in every cell its tolerance
ball reaches — up to eight, for a vertex near a corner — makes the lookup a single-cell
scan while keeping the weld correct. The alternative, scanning the neighbourhood at lookup
time, costs eight scans per query instead of a bounded amount of extra registration.

The tolerance is per-axis, derived as half a cell on each axis but never larger than a
fixed absolute epsilon, so a small collector over a small box does not get an absurdly
tight tolerance and a huge one does not get an absurdly loose one. The tolerance ball must
stay smaller than a cell, or the eight-cell registration is not enough.

## `Collector.remove_duplicate_faces`

**Contract** — drops faces that name the same three vertices with the same payload, in any
winding. Quadratic in the face count, and the survivor is chosen arbitrarily (the last face
is swapped into the removed slot), so face order is not preserved.

**Notes** — "the same in any winding" means all six permutations are compared, so a face
and its reverse both count as duplicates of each other. That is right for a collision soup,
where a doubled wall from two co-located brushes should become one surface regardless of
which way each was wound — but it also means a genuinely two-sided pair of faces collapses
to one, which is only harmless because facing is decided by the query rather than by the
triangle (see [`xrCDB_ray.cpp`](xrCDB_ray.cpp.md)).

## `Collector.edge_adjacency`

**Contract** — produces, for every face and every one of its three edges, the index of the
face sharing that edge, or a sentinel meaning *open edge*. The output is a flat array of
`3 x face_count` entries indexed by `face * 3 + edge`.

```text
FUNCTION edge_adjacency() -> list<int>
  edges <- for each face f and each edge e of f, the record
             (face = f, edge = e, a = min(endpoints), b = max(endpoints))
  sort edges by (a, b, face)
  result <- all entries set to none
  FOR EACH adjacent pair (i, j) in the sorted order
    IF i.a == j.a AND i.b == j.b
      result[i.face * 3 + i.edge] <- j.face
      result[j.face * 3 + j.edge] <- i.face
  RETURN result
```

**Normalizing each edge to (smaller, larger) before sorting is what makes this work**: two
faces sharing an edge traverse it in opposite directions, so the unordered key is the only
thing they agree on. Sorting then puts the two halves of every shared edge next to each
other, which turns a quadratic pairwise search into one sort and one linear pass — the
source keeps the quadratic version alongside as a commented-out oracle, which is a good
habit to keep: it is the only check the fast version ever had.

An edge shared by *three or more* faces — which happens in authored level geometry — links
only one pair, whichever two land adjacent under the sort's tiebreak on face index. That is
a silent inaccuracy, not a guarded case, and any consumer of adjacency must tolerate it.

## Notes

The packed collector carries a **per-face flag word** that the triangle itself has no room
for and that nothing in this module reads or writes — it is passed in by the producer and
read back by the producer. It is storage the collector provides on behalf of a level
compiler that does not ship in this repository, so what the bits mean **could not be
recovered here**. A rebuild targeting only the runtime can drop it.
