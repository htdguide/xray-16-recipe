# src/xrCore/_cylinder.cpp

> Ray against a finite capped cylinder, with the two degenerate orientations — parallel to the axis and perpendicular to it — handled as separate cases because the general solution divides by zero in both.

**Needs** — [`_cylinder.h`](_cylinder.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md) · [`log.h`](log.h.md)

**Used by** — [`_cylinder.h`](_cylinder.h.md)

**Tier floor** — T2: pure floating-point arithmetic over a value type. Held off T3 only by running inside picking loops.

## Purpose

A cylinder is three surfaces glued together — two discs and a tube — and a ray can enter
through any of them and leave through any other. The whole file is one routine that enumerates
those cases, plus a thin wrapper that reduces the answer to "nearest hit, if it beats the
distance I already have".

## State

`Stateless.` Everything is computed from the cylinder and the ray.

## `intersect(start, direction, out_parameters, out_codes)` — the full form

**Contract** — Takes a ray origin and direction in world space, and writes up to two ray
parameters with a surface code for each. Returns how many were written — 0, 1 or 2. The
parameters are distances along the *given* direction, so a unit direction yields world
distances. Pure, allocation-free, no early distance culling: this reports every intersection
of the infinite line, including ones behind the origin, and it is the caller's job to
discard those. Thread-safe against a shared immutable cylinder.

**Invariants**

- The returned parameters are **not sorted**, and in the general case the first is whichever
  surface happened to be tested first — a cap before a wall. A caller that wants the nearer
  hit must compare; the narrowing form below does.
- The surface code array is written in lockstep with the parameter array. Slot `i` of one
  always describes slot `i` of the other.
- Negative parameters are returned, meaning the intersection is behind the ray origin. Two
  returned parameters of opposite sign is exactly the signature of an origin *inside* the
  cylinder, and the narrowing form reads it that way.

### The working frame

Every case works in a frame where the cylinder's axis is the third coordinate. That turns
the tube into a circle test in the first two coordinates and the caps into two constant
planes in the third, which is what makes the case analysis tractable at all.

```text
FUNCTION to_cylinder_frame(cylinder, start, direction)
  # Build an orthonormal basis whose third axis is the cylinder's axis. The
  # other two are arbitrary -- nothing downstream cares which way they point,
  # only that they are orthonormal, because the answer is a distance.
  (u, v, w) = orthonormal_basis_around(cylinder.direction)

  d = (dot(u, direction), dot(v, direction), dot(w, direction))
  direction_length = magnitude(d)
  d = d / direction_length                 # d is now unit in the local frame

  offset = start - cylinder.centre
  p = (dot(u, offset), dot(v, offset), dot(w, offset))

  RETURN (p, d, 1 / direction_length)
```

**Notes** — The direction is normalized *inside* the frame and its original length is kept
as a reciprocal. Every parameter produced downstream is multiplied by that reciprocal on the
way out, which is what makes the returned values distances along the caller's direction
rather than along the normalized one. A rebuild that forgets the rescale returns answers
that are correct only for unit inputs — and most callers do pass unit directions, so the bug
hides.

### Case one — the ray runs along the axis

```text
IF |d.z| >= 1 - epsilon THEN                 # parallel to the axis
  IF p.x² + p.y² <= radius² THEN
    # It enters one cap and leaves the other. Both hits are caps; there is
    # no wall hit at all, which is why this case cannot fall through to the
    # general one (the wall quadratic would have a zero leading coefficient).
    scale = reciprocal_direction_length / d.z
    out[0] = (+half_height - p.z) * scale ; code[0] = cap
    out[1] = (-half_height - p.z) * scale ; code[1] = cap
    RETURN 2
  RETURN 0                                    # runs alongside, outside the tube
```

**Notes** — The epsilon is one part in a million million, which is far tighter than any
other tolerance in the geometry layer. It is tight deliberately: this branch is not an
approximation of the general case but a *replacement* for it, and widening the epsilon
starts routing nearly-parallel rays — where the general case is still perfectly
well-conditioned — into a branch that reports two cap hits and no wall hit.

### Case two — the ray is perpendicular to the axis

```text
IF |d.z| <= epsilon THEN
  IF |p.z| > half_height THEN RETURN 0        # the slab misses entirely
  # Only the wall can be hit; the caps are parallel to the ray.
  RETURN solve_wall_quadratic(p, d, radius, reciprocal_direction_length)
```

**Invariants** — The height test comes first and is exact: a perpendicular ray either lies
between the caps for its whole length or never touches the cylinder.

The wall quadratic is the ordinary circle-versus-line solve in the first two coordinates:

```text
FUNCTION solve_wall_quadratic(p, d, radius) -> roots
  a = d.x² + d.y²
  b = p.x*d.x + p.y*d.y
  c = p.x² + p.y² - radius²
  discriminant = b² - a*c
  IF discriminant < 0 THEN RETURN none            # misses the tube
  IF discriminant = 0 THEN RETURN one root at -b/a    # tangent
  root = sqrt(discriminant)
  RETURN two roots at (-b - root)/a and (-b + root)/a
```

### Case three — the general orientation

The ray crosses both cap planes and may or may not cross the tube. Test the caps first,
because a ray through both caps cannot touch the wall and the test is cheap.

```text
count = 0
FOR EACH cap_plane IN (+half_height, -half_height)
  t = (cap_plane - p.z) / d.z
  hit = (p.x + t*d.x, p.y + t*d.y)
  IF hit.x² + hit.y² <= radius² THEN
    code[count] = cap ; out[count] = t * reciprocal_direction_length ; count += 1

IF count = 2 THEN RETURN 2         # in one cap and out the other; no wall hit

# One cap hit means exactly one wall hit; no cap hit means zero or two.
roots = solve_wall_quadratic(p, d, radius)
FOR EACH root IN roots               # near root first, then far
  IF root lies between the two cap parameters THEN
    code[count] = wall ; out[count] = root * reciprocal_direction_length ; count += 1
  IF count = 2 THEN BREAK
RETURN count
```

**Invariants**

- A wall root is accepted only if it lies **between** the two cap-plane parameters. That
  interval test is what makes the cylinder finite: the quadratic solves the infinite tube,
  and the cap parameters are the clip.
- The two cap parameters are not sorted — which of the two is smaller depends on the sign of
  the axis component of the direction — so the interval test is written for both orderings.
  A rebuild sorts them once and tests one interval; the original tests the ordering at every
  root, which is the same decision spelled three times.
- The tangent case (discriminant exactly zero) yields one wall root and is handled; it is
  measure-zero and the code path exists only so a tangent ray is not silently reported as a
  miss while a cap hit is already recorded.

**Notes** — This routine is the only place in the geometry layer that *logs* on a failed
assertion rather than just asserting: a zero-length transformed direction prints the ray and
the basis before it fails. That is a debugging aid for the one input that produces garbage
without producing a fault — a caller that passes a zero direction — and it survives here
because the failure was hard enough to find once.

## `intersect(start, direction, distance)` — the narrowing form

**Contract** — Runs the full form, then reduces the result to the three-way classification
shared with the sphere and the box, narrowing the caller's current best distance in place.
Returns "no hit" both when the line misses and when every hit is farther than the distance
already held.

```text
FUNCTION nearest_hit(start, direction, best_distance) -> classification
  (count, parameters) = intersect(start, direction)
  IF count = 0 THEN RETURN no_hit

  origin_inside = false
  narrowed = false
  FOR EACH t IN the returned parameters
    IF t < 0 THEN
      # A hit behind the origin. With two hits that means the origin lies
      # between them -- inside the cylinder. With one it means the line only
      # grazes behind us.
      IF count = 2 THEN origin_inside = true
      CONTINUE
    IF t < best_distance THEN
      best_distance = t
      narrowed = true

  IF NOT narrowed THEN RETURN no_hit
  IF origin_inside THEN RETURN origin_inside_classification
  RETURN origin_outside_classification
```

**Invariants** — The surface codes are computed and then discarded by this form. That is a
real loss of information: a caller that wants the surface normal at the hit must use the full
form. The two forms exist because picking wants the distance and shading wants the surface.

**Notes** — "Origin inside" is inferred from a negative parameter alongside a second hit,
not from a containment test. That inference is correct for a convex shape and is how the
sphere and the oriented box decide the same question — worth preserving as a shared idea
rather than reimplementing per shape.
