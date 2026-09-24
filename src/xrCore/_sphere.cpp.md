# src/xrCore/_sphere.cpp

> The smallest enclosing sphere of a point set, by Welzl's algorithm — this is what turns a mesh's vertices into the bounding sphere that visibility and collision cull against.

**Needs** — [`_sphere.h`](_sphere.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`../xrCommon/xr_list.h`](../xrCommon/xr_list.h.md)
**Used by** — [`_sphere.h`](_sphere.h.md)
**Tier floor** — T2: it is numerical work over a list of points, done once per mesh at load time. The vectors' layout matters for speed, not for correctness.

## Purpose

Every renderable and every collision shape needs a bounding sphere, and a loose one costs visibility tests all frame. This computes the *minimal* one. It runs once per mesh at load time, so the algorithm is chosen for tightness rather than speed.

## The idea

The minimal enclosing sphere of a point set in three dimensions is determined by at most four of its points — its *support set* — all lying on the surface. Welzl's algorithm finds that set by moving points that violate the current sphere to the front of the list and recursing, so that the points that determine the answer migrate to the front and are found quickly on later passes. The engine uses the pivoting variant, which picks the single worst violator each round instead of the first.

## State

```text
RECORD Basis                       # the current sphere plus how it was determined
  count          : int             # how many points define it, 0..4
  support_count  : int             # size of the support set
  origin         : (real,real,real)# the first pushed point; everything is relative to it
  centre         : list<(real,real,real)>   # 4 entries: the centre after each push
  squared_radius : list<real>               # 4 entries, matching
  basis_vectors  : list<(real,real,real)>   # 4 entries: the Gram-Schmidt basis
  gram_norms     : list<real>               # 4 entries: 2 * squared length of each basis vector
  projections    : list<real>               # 4 entries: coefficients against earlier basis vectors
  shift          : list<real>               # 4 entries: how far the centre moved on each push

RECORD Solver
  points      : list<(real,real,real)>   # a linked list: elements are spliced, never copied
  basis       : Basis
  support_end : position in points       # everything before it is the support set
```

**Invariants**

- The support set is a **prefix** of the point list. "Moving to front" is how membership is recorded; there is no separate set.
- The basis holds at most four entries, because four points determine a sphere in three dimensions. The empty sphere is represented by a negative squared radius, which makes every point's excess positive and is what starts the algorithm.
- The point list is a linked list specifically so that splicing a point to the front is constant-time and does not invalidate the positions the recursion holds.

## `compute`

**Contract** — Takes a point array and produces the smallest enclosing sphere. Pure with respect to its input; builds its own working list. The radius is the square root of the squared radius the solver carries, taken once at the end.

```text
FUNCTION compute(points) -> Sphere
  solver = new Solver
  FOR EACH p IN points  DO  solver.add(p)
  solver.build()
  RETURN (solver.centre, sqrt(solver.squared_radius))
```

## `build` — the pivoting search

```text
FUNCTION build() -> void
  basis.reset()                          # empty sphere: squared radius = -1
  support_end = start of points
  pivot_search(end of points)

FUNCTION pivot_search(limit) -> void
  t = second element of points
  move_to_front_search(t)
  previous_radius = 0
  REPEAT
    worst = the point in [t, limit) with the largest excess over the current sphere
    IF the largest excess is not positive THEN BREAK      # every point is enclosed
    t = support_end
    IF t is the worst point THEN advance t                # do not re-examine it
    previous_radius = basis.squared_radius
    basis.push(worst)
    move_to_front_search(support_end)
    basis.pop()
    move_to_front(worst)
  UNTIL the excess is not positive OR the radius did not grow
```

**Invariants** — The loop terminates only when either no point violates the sphere *or the radius stopped growing*. The second condition is the guard against the degenerate cases — coincident and nearly-coplanar points — where floating-point error would otherwise let it cycle. This is the practical termination criterion and a rebuild must keep it.

```text
FUNCTION move_to_front_search(limit) -> void
  support_end = start of points
  IF basis.count == 4 THEN RETURN        # already fully determined
  FOR EACH p IN points BEFORE limit
    IF excess(p) > 0 THEN
      IF basis.push(p) THEN
        move_to_front_search(position of p)   # recurse over the earlier prefix
        basis.pop()
        move_to_front(p)                      # p joins the support prefix
```

**Notes** — `move_to_front` splices the point to the head of the list and, when it was the support boundary, advances the boundary — so the support set grows by adopting the point that was just found to matter. The list ordering *is* the algorithm's memory, and this is why the structure must be a splice-capable list rather than an array.

## `push` — adding a point to the basis

**Contract** — Extends the current sphere to pass through one more point, by Gram-Schmidt orthogonalization against the points already in the basis. Returns whether the point was accepted; a rejected point is numerically degenerate with the existing basis and must be skipped.

```text
FUNCTION push(p) -> bool
  IF the basis is empty THEN
    origin = p; centre[0] = p; squared_radius[0] = 0
    RETURN true

  v[m] = p - origin
  # Project out the components along the earlier basis vectors.
  FOR i FROM 1 TO m-1
    projection[m][i] = 2 * dot(v[i], v[m]) / gram_norm[i]
  FOR i FROM 1 TO m-1
    v[m] = v[m] - projection[m][i] * v[i]
  gram_norm[m] = 2 * squared_magnitude(v[m])

  # Reject a point that adds no independent direction. The threshold is
  # relative to the current squared radius, so it scales with the model --
  # an absolute epsilon would reject valid points on large meshes and
  # accept degenerate ones on small meshes.
  IF gram_norm[m] < 1e-16 * current_squared_radius THEN RETURN false

  excess = squared_distance(p, centre[m-1]) - squared_radius[m-1]
  shift[m] = excess / gram_norm[m]
  centre[m] = centre[m-1] + shift[m] * v[m]
  squared_radius[m] = squared_radius[m-1] + excess * shift[m] / 2
  m = m + 1
  RETURN true
```

**Invariants** — `excess` is the amount by which the new point lies outside the current sphere; the centre moves along the newly orthogonalized direction by exactly enough to reach it, and the radius grows by half the product. Popping is a decrement — every intermediate sphere is kept, so unwinding the recursion is free.

**Notes** — The rejection threshold `1e-16` relative to the squared radius is the one constant here that is neither derived nor explained in the source. It is at the edge of double precision and the arithmetic is single-precision, which suggests it is effectively "reject only exact degeneracies". A rebuild should treat it as a tuning value to be re-derived for its own precision, not as a number to copy.

## `excess`

**Contract** — How far outside the current sphere a point lies, in squared distance: squared distance to the centre minus the squared radius. Positive means outside. With the empty sphere's radius set to -1, every point has positive excess, which is what bootstraps the search.
