# src/Common/NvMender2003/NVMeshMender.cpp

> Computes a per-vertex tangent basis for normal mapping, splitting vertices wherever a crease means one basis cannot serve two adjacent triangles.

**Needs** — [`NVMeshMender.h`](NVMeshMender.h.md) · [`../d3d9compat.hpp`](../d3d9compat.hpp.md)
**Used by** — [`NVMeshMender.h`](NVMeshMender.h.md)
**Tier floor** — T2: it is float arithmetic over arrays with no layout, latency or device constraint. It runs at asset-build time, not per frame.

## Purpose

Per-pixel lighting needs a coordinate frame at every vertex that maps the normal map's
texture directions onto the surface: a tangent along increasing horizontal texture
coordinate, a binormal along increasing vertical, and the normal. This file computes all
three from geometry and texture coordinates alone.

The hard part is not the arithmetic — that is twenty lines — but the *smoothing decision*.
Averaging a frame across a sharp crease produces a visible smear; refusing to average
anywhere produces faceting. The answer here is: average across an edge only when the two
triangles' frames are close enough, and where they are not, **duplicate the shared vertex**
so each side gets its own. That is why the pass can return more vertices than it was given,
and why it must also return a mapping from new vertices back to old ones.

This is a vendored third-party file, lightly adapted: its allocator and containers were
swapped for the engine's, and its one dependency on a graphics library's vector maths was
replaced with local arithmetic. Its structure and comments are the vendor's.

## State

```text
RECORD Vertex                       # the generator's own vertex; supplied and returned
  position  : three real            # caller fills
  texture_s : real                  # caller fills
  texture_t : real                  # caller fills
  normal    : three real            # caller fills only when not asking for computed normals
  tangent   : three real            # computed
  binormal  : three real            # computed

RECORD Triangle
  corners   : three int             # indices into the vertex list; REWRITTEN by the pass
  normal    : three real            # per-face, unnormalized when area-weighted
  tangent   : three real            # per-face
  binormal  : three real            # per-face
  group     : optional<int>         # which smoothing group this face joined, this pass
  handled   : bool                  # visited during this pass's group building
  id        : int                   # its own index, so a group can name it

RECORD MenderState
  triangles          : list<Triangle>
  vertex_children    : map<position, list<triangle id>>
                                    # every triangle touching a given POSITION — not a
                                    # given vertex index. This is the central structure.
  original_count     : int          # vertex count before any splitting
  normal_threshold   : real         # cosine; see thresholds below
  tangent_threshold  : real
  binormal_threshold : real
  area_weight        : real         # 0..1
  respect_splits     : bool
```

**Invariants**

- `vertex_children` is keyed by **position**, not by vertex index. That is the pass's
  central decision: a mesh usually already contains several vertices at one position,
  duplicated because they differ in texture coordinate, and a naive adjacency built on
  indices would see them as unrelated and smooth nothing. Keying on position reunites them.
  The `respect_splits` mode turns this off for callers who know their existing splits are
  meaningful.
- The position key is compared **exactly**, not within a tolerance. The comment is explicit
  about why: the key must find the same position reliably, and a tolerance-based comparison
  is not a valid ordering — it is not transitive, and a map built on it misplaces entries.
  The consequence is that positions differing in the last bit are different vertices and are
  never smoothed together. A rebuild must either keep exact comparison or quantize positions
  to a grid *before* keying, never compare fuzzily.
- The triangle count never changes. Only vertices are added and only corner indices are
  rewritten.
- The new-to-old mapping always resolves to an *original* vertex, never to another new one,
  even when a new vertex is split again.

## The thresholds

**Contract** — three thresholds, each the **cosine** of the maximum angle across which that
vector may be smoothed, ranging from −1 (smooth everything) to +1 (smooth nothing).

- **normal threshold** — the classic crease angle. Ignored when the caller supplies normals.
- **tangent threshold** and **binormal threshold** — the same test applied to the texture
  directions, which is what catches a *texture* seam that is not a geometric crease: two
  triangles lying flat against each other but mapped from opposite sides of a mirrored
  texture have opposed tangents and must not be averaged.

**area weight** blends between two ways of computing a face normal: at 0 the face normal is
normalized before it is averaged, so every face votes equally; at 1 it is left unnormalized,
so its magnitude — twice the triangle's area — weights the vote. Large faces then dominate.

**Invariants** — the defaults the generator sets for itself are 0.3 for normals (about 72°)
and 0 for the two texture directions (90°, so anything past perpendicular splits), with no
area weighting. The engine's call sites pass their own.

## `Mend`

**Contract** — the single entry point. Takes the vertex list, the index list and an empty
new-to-old mapping, and rewrites all three in place. Returns success; there is no failure
path, and a malformed input is caught by debug assertions rather than reported.

```text
FUNCTION mend(vertices, indices, new_to_old, thresholds, area_weight,
              compute_normals, respect_splits, fix_cylindrical) -> bool
  IF fix_cylindrical
    repair_wrap_seams(vertices, indices, new_to_old)   # must run first: it changes
                                                        # texture coordinates, which every
                                                        # later step reads
  build_state(vertices, indices, new_to_old, compute_normals)

  FOR EACH position, touching_triangles IN vertex_children
    IF compute_normals
      process(NORMALS,   touching_triangles, vertices, new_to_old, position)
    process(TANGENTS,    touching_triangles, vertices, new_to_old, position)
    process(BINORMALS,   touching_triangles, vertices, new_to_old, position)

  write_corner_indices_back(indices)
  orthogonalize(vertices)
  RETURN true
```

**Invariants** — the three passes run per position and in this order, and each is independent:
a position may be split for its normals and not for its tangents, or the reverse. They
accumulate — a split made by the normal pass is visible to the tangent pass, because both
read the same triangle corner indices.

## `build_state`

**Contract** — initializes the mapping to the identity, records the original vertex count,
builds one triangle record per index triple with its face vectors, and builds the
position-to-triangles map.

```text
FUNCTION build_state(vertices, indices, new_to_old, compute_normals)
  new_to_old <- [0, 1, 2, ... count(vertices) - 1]
  original_count <- count(vertices)
  FOR EACH triple IN indices                 # asserted to be a multiple of three
    t <- a triangle with those three corners
    compute_face_vectors(t, vertices, compute_normals)
    t.id <- its index
    append t TO triangles
  FOR EACH t IN triangles
    FOR EACH corner IN t
      append t.id TO vertex_children[position of that corner]
```

## `compute_face_vectors`

**Contract** — computes one triangle's normal, tangent and binormal.

```text
FUNCTION compute_face_vectors(t, vertices, compute_normals)
  IF compute_normals
    e0 <- position(corner 1) - position(corner 0)
    e1 <- position(corner 2) - position(corner 0)
    n  <- cross(e0, e1)                      # magnitude is twice the triangle's area
    IF area_weight < 1
      t.normal <- normalize(n) * (1 - area_weight) + n * area_weight
    ELSE
      t.normal <- n
  (t.tangent, t.binormal) <- texture_gradients(corner 0, corner 1, corner 2)
```

## `texture_gradients`

**Contract** — solves for the two directions in object space along which the texture coordinates
increase. This is the arithmetic core of the whole file.

Given edges `P = v1 − v0` and `Q = v2 − v0` and the corresponding texture-coordinate
differences `(s1, t1)` and `(s2, t2)`, the two unknown directions satisfy

```text
P = s1·T + t1·B
Q = s2·T + t2·B
```

which inverts to

```text
FUNCTION texture_gradients(v0, v1, v2) -> (tangent, binormal)
  P  <- v1.position - v0.position
  Q  <- v2.position - v0.position
  s1 <- v1.s - v0.s ;  t1 <- v1.t - v0.t
  s2 <- v2.s - v0.s ;  t2 <- v2.t - v0.t

  determinant <- s1*t2 - s2*t1
  IF |determinant| <= 0.0001
    scale <- +1 IF determinant > 0 ELSE -1    # degenerate: keep the sign, drop the scale
  ELSE
    scale <- 1 / determinant

  tangent  <- (t2*P - t1*Q) * scale
  binormal <- (s1*Q - s2*P) * scale
```

**Invariants** — a determinant near zero means the triangle has no area *in texture space* —
its three corners share a texture coordinate, or lie on a line in it. There is no tangent
frame to find. The degenerate branch keeps the direction's sign and abandons its magnitude,
producing a frame that is wrong but finite and consistently oriented; the final
orthogonalization then repairs it. Dividing anyway would produce infinities that propagate
through every averaged neighbour.

The threshold is absolute, not relative to the triangle's scale, which means a very small
but well-formed triangle can be treated as degenerate. That is a real limitation of the
constant and a rebuild working at a different world scale should re-derive it.

**Notes** — the results are *unnormalized* and deliberately so: their magnitudes weight the
averaging that follows, exactly as the area-weighted normal does.

## `process` — one smoothing pass at one position

**Contract** — given every triangle touching one position and a choice of which vector to smooth,
partition those triangles into smoothing groups, compute one averaged vector per group, and
ensure no two groups share a vertex at this position — splitting a vertex where they would.

```text
FUNCTION process(which_vector, touching, vertices, new_to_old, position)
  FOR EACH t IN touching
    t.group <- none ;  t.handled <- false      # groups are per-pass, per-position

  groups <- empty
  FOR EACH t IN touching
    IF NOT t.handled
      build_groups(t, touching, groups, vertices, which_vector)

  FOR EACH g IN groups                          # the averaged vector for each group
    averaged[g] <- normalize(sum of which_vector over the triangles of g)

  claimed <- empty set of vertex indices
  FOR EACH g IN groups
    mine <- empty set
    FOR EACH t IN g, FOR EACH corner IN t
      IF position of corner = position
        IF corner index IS IN claimed          # another group already owns this vertex
          new <- a copy of that vertex with which_vector <- averaged[g]
          append new TO vertices
          record_mapping(corner index, new_to_old)
          rewrite that index to the new one, THROUGHOUT THIS GROUP ONLY
        ELSE
          set which_vector of that vertex <- averaged[g]
        add corner index TO mine
    claimed <- claimed UNION mine               # only after the group is finished
```

**Invariants**

- The first group to reach a vertex keeps it; every later group that wants it gets a copy.
  Which group is "first" depends on iteration order, so the *identity* of the surviving
  vertex is arbitrary — but the resulting geometry is not, because all copies are
  equivalent.
- The claimed set is updated only **after** a whole group is processed, not as each vertex
  is taken. Within one group a vertex may be touched by several triangles and must not be
  split from itself.
- The rewrite is confined to the current group. Triangles in other groups keep pointing at
  the original vertex, which is exactly the separation being created.

## `build_groups` — growing one smoothing group

**Contract** — depth-first growth from one triangle through its neighbours, joining a neighbour's
existing group when the vectors are close enough and starting a new group otherwise.

```text
FUNCTION build_groups(t, touching, groups, vertices, which_vector)
  IF t is none OR t.handled
    RETURN
  (n1, n2) <- find_neighbours(t, touching, vertices)   # at most two

  IF n1 exists AND n1.group exists AND can_smooth(t, n1, which_vector)
    join t to n1.group
  IF n2 exists AND n2.group exists AND can_smooth(t, n2, which_vector)
    join t to n2.group                                 # note: may OVERRIDE the first join
  IF t.group is none
    t.group <- a fresh group containing only t
  t.handled <- true

  build_groups(n1, ...)                                # grow outward
  build_groups(n2, ...)
```

**Invariants and known limitations**

- A triangle that can smooth with *both* neighbours ends up recorded in both groups'
  membership lists but names only the second as its own group. Its vector therefore
  contributes to two averages while it receives one. This is a defect in the vendor code,
  not a decision; it makes the result slightly order-dependent at positions where three or
  more faces meet smoothly. A rebuild should merge the two groups instead.
- At most two neighbours are considered, because the triangles under consideration all
  share one position and a manifold surface gives each of them exactly two edge-neighbours
  in that fan. A non-manifold position with three or more faces on an edge silently loses
  the rest.
- Recursion depth is bounded by the number of triangles at one position — small in
  practice, but unbounded in principle on pathological meshes.

## `can_smooth`

**Contract** — compare the chosen vectors of two triangles. Normalize both (they may be
area-weighted or unnormalized), take their dot product, and admit smoothing when it meets
the threshold. Two *zero* vectors are also admitted, unconditionally.

**Invariants** — the zero-vector exception is what keeps degenerate triangles from fragmenting
their neighbourhood: a triangle with no area has no meaningful direction, so it is allowed
to join whatever it touches rather than forcing a split. Note it only applies when *both*
are zero.

## `find_neighbours` and edge sharing

**Contract** — two triangles are neighbours when they share an edge. Two definitions of "share"
exist and the mode chooses:

- **by position** (the default) — an edge is a pair of positions, so triangles joined at a
  texture seam still count as neighbours. This is what makes smoothing work across an
  existing split.
- **by index** — an edge is a pair of vertex indices, so an existing split is respected and
  never smoothed across.

**Notes** — the edge test compares the three directed edges of one triangle against all three
edges of the other in either direction. It is a six-way comparison written out, and it
decides adjacency for every pair of faces at every position, so it is the pass's inner
loop. The vendor's own note says a full adjacency map built once would be the obvious
optimization and that trying it did not help — worth knowing before a rebuild optimizes it.

## `orthogonalize` — the final repair

**Contract** — runs once over every vertex after all smoothing. Makes the tangent and binormal
perpendicular to the final smoothed normal and to each other, normalizes them, and
manufactures a valid frame wherever the result is degenerate.

```text
FUNCTION orthogonalize(vertices)
  FOR EACH v IN vertices
    # Gram-Schmidt against the final normal
    T <- v.tangent  - dot(v.normal, v.tangent) * v.normal
    B <- v.binormal - dot(v.normal, v.binormal) * v.normal - dot(T, v.binormal) * T
    v.tangent  <- normalize(T)
    v.binormal <- normalize(B)

    IF length(v.tangent) > 0.5 AND length(v.binormal) > 0.5
      IF dot(v.binormal, v.tangent) > 0.999        # nearly parallel despite the above
        v.binormal <- cross(v.normal, v.tangent)
      RETURN
    IF length(v.tangent) > 0.5                     # tangent survived, rebuild binormal
      v.binormal <- cross(v.normal, v.tangent)
    ELSE IF length(v.binormal) > 0.5               # binormal survived, rebuild tangent
      v.tangent  <- cross(v.binormal, v.normal)
    ELSE                                           # neither survived: invent a frame
      axis <- whichever of the x and y axes is FURTHER from the normal
      v.tangent  <- cross(v.normal, axis)
      v.binormal <- cross(v.normal, v.tangent)
```

**Invariants**

- The normal is trusted absolutely — it is asserted non-zero — and the other two are made to
  fit it. That is the right way round: the normal comes from geometry, the other two from
  texture coordinates, which are the less reliable input.
- The fabricated frame is *valid but arbitrary*. The surface will light consistently and the
  normal map will be applied in a direction unrelated to its authoring. This is a
  last-resort branch for vertices whose texture mapping carries no direction at all, and it
  is better than a zero frame, which would black out the pixel.
- Choosing the axis *further* from the normal is what keeps the first cross product from
  being near-zero.
- The near-parallel repair after a successful orthogonalization catches cases the
  Gram-Schmidt step leaves almost degenerate; 0.999 corresponds to about 2.5°.

**Notes** — the vendor's comment wonders aloud whether this should run before smoothing rather
than after. It runs after, which is correct: orthogonalizing against a per-face normal and
then averaging does not produce a frame orthogonal to the averaged normal.

## `repair_wrap_seams`

**Contract** — optional, and runs before anything else. Repairs the seam that cylindrical texture
projection leaves, where a triangle's texture coordinates run 0.9 → 0.0 instead of
0.9 → 1.0. Handles the two texture axes independently.

```text
FUNCTION repair_wrap_seams(vertices, indices, new_to_old)
  FOR EACH triangle IN indices
    duplicated <- empty set of corners
    FOR EACH edge (begin -> end) OF the triangle          # all three, cyclically
      FOR axis IN (s, t)
        a <- axis of vertex at begin ;  b <- axis of vertex at end
        IF a AND b are both within 0..1 AND |a - b| > 0.5
          corner <- whichever of begin, end has the SMALLER coordinate
          IF corner NOT IN duplicated
            duplicate that vertex, add 1 to its coordinate on this axis,
            point this triangle's corner at the duplicate,
            record_mapping(original index, new_to_old),
            add corner TO duplicated
          ELSE
            add 1 to the already-duplicated vertex's coordinate on this axis
```

**Invariants**

- Only coordinates already inside 0..1 are considered, and a difference greater than half
  the range is taken as evidence of a wrap. Both are stated limits: a mesh whose texture
  coordinates legitimately span more than one repeat will be mangled by this mode, which is
  why it is off by default and the vendor warns against enabling it globally.
- The duplication is per triangle, and the set of already-duplicated corners is reset for
  each triangle — so a vertex shared by several wrapping triangles is duplicated once per
  triangle. That is intended: each triangle needs its own continuation of the coordinate.
- The second branch handles a corner that wrapped on both axes, adding 1 to the second axis
  of the vertex already made for the first.

## `record_mapping`

**Contract** — appends one entry to the new-to-old mapping when a vertex is duplicated, resolving
transitively so that the entry always names an *original* vertex.

```text
FUNCTION record_mapping(source_index, original_count, new_to_old)
  IF source_index >= original_count            # we are copying a copy
    append new_to_old[source_index]            # inherit its ancestor
  ELSE
    append source_index
```

**Invariants** — this is what makes the mapping usable by the caller in one indexing step rather
than a chase. It is also what
[`mender_input_output.h`](mender_input_output.h.md) relies on to recover per-vertex data
the generator does not carry.

## Notes

The file replaces the graphics library's vector-normalize with a local one, explicitly to
avoid a runtime dependency on that library. The local version **does not guard against a
zero-length vector** — it divides by the reciprocal square root unconditionally — which is
why the zero-vector cases are handled by the callers' explicit checks instead. A rebuild
should make normalize safe and simplify those callers.

Positions are ordered for the map by comparing x, then y, then z, exactly. See the
invariant above: this is a deliberate refusal of tolerance-based comparison, and it is the
right call.

The pass is asset-build work. Running it per level load would be visible; the engine runs
it in its mesh tools.
