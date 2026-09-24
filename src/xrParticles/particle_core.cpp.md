# src/xrParticles/particle_core.cpp

> The domain: eleven geometric regions sharing one record, each with a rule for drawing a
> uniform sample, a rule for testing containment, and a rule for being moved by a matrix.

**Needs** — [`particle_core.h`](particle_core.h.md) · [`psystem.h`](psystem.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md) · [`xrCore/_random.h`](../xrCore/_random.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: one record is reinterpreted eleven ways and is read from authored files
as raw bytes; nothing above T1 can promise that layout.

## Purpose

Every particle that is born gets its position, velocity, size, rotation and colour by drawing
a sample from a domain. Every particle that is killed, avoided or bounced is tested against a
domain. So the domain is the single geometric vocabulary of the whole chapter, and the
*sampling distribution* of each kind is the part a rebuild most easily gets wrong: a sphere
sampled uniformly by volume instead of by the rule below produces a visibly different effect
from the same authored file.

The file is separate from the actions because domains are shared by twenty of the thirty-one
actions and by the authoring format. It carries no behaviour of its own beyond geometry.

## State

One record, reused for every kind. The field names are positional, not semantic: what `p1`
means depends on the kind tag. This is the cost of the frozen layout, and the mapping below
is the only place it is written down.

```text
RECORD Domain
  kind        : int (32-bit)   # DomainKind, see psystem.h
  p1, p2      : vec3
  u, v        : vec3
  radius1     : real
  radius2     : real
  radius1Sqr  : real
  radius2Sqr  : real
# 68 bytes, fields in exactly this order, four-byte alignment, little-endian.
```

Per-kind field meaning, and the invariants the constructor establishes:

| Kind | `p1` | `p2` | `u`, `v` | `radius1` | `radius2` | `radius1Sqr` | `radius2Sqr` |
|---|---|---|---|---|---|---|---|
| Point | the point | — | — | — | — | — | — |
| Line | one end | **delta** to the other end | — | — | — | — | — |
| Box | min corner | max corner | — | — | — | — | — |
| Plane | a point on it | unit normal | — | plane offset `-(p1·p2)` | — | — | — |
| Triangle | first vertex | unit normal | edges to the other two vertices | plane offset | — | length of `u` | length of `v` |
| Rectangle | corner | unit normal | the two edge vectors | plane offset | — | length of `u` | length of `v` |
| Sphere | centre | — | — | outer radius | inner radius | outer² | inner² |
| Cylinder | base centre | **delta** to the far end | orthonormal frame across the axis | outer radius | inner radius | outer² | `1/(p2·p2)` |
| Cone | apex | **delta** to the base centre | orthonormal frame across the axis | outer radius at the base | inner radius at the base | outer² | `1/(p2·p2)` |
| Disc | centre | unit normal | orthonormal frame in the plane | outer radius | inner radius | plane offset | — |
| Blob | centre | — | — | standard deviation | `1/(sigma·sqrt(2·pi))` | — | `-0.5/sigma²` |

Invariants:

- Box stores its corners sorted per axis, so `p1 ≤ p2` componentwise regardless of the order
  the author gave them.
- Sphere, cylinder, cone and disc store the larger of the two authored radii in `radius1`, so
  `radius1 ≥ radius2` always and the shell between them is well-defined. An authored inner
  radius of zero makes a solid region.
- Plane-like kinds (plane, triangle, rectangle, disc) store a *unit* normal and the plane's
  offset, so signed distance is one dot product and one add. Disc is the odd one out: it keeps
  its offset in `radius1Sqr` rather than `radius1`, because `radius1` is already the outer
  radius. Every caller must know this; it is the single most error-prone field in the module.
- Cylinder and cone keep the reciprocal of their axis length squared, so the axial projection
  of a point is a dot product and a multiply with no division. A zero-length axis stores zero
  there, which makes every point project to the base and be rejected rather than divide by
  zero.
- The frame `u`, `v` for cylinder, cone and disc is *derived*, not authored: it is any
  orthonormal pair across the axis. The derivation below must be reproduced exactly, because
  it fixes where angle zero points, and a source that samples a ring is visibly rotated if the
  frame differs.

**Notes** — these derived fields are not recomputed at load time. An authored file stores the
whole record, derived fields included, and the loader trusts them
([`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md)). A rebuild
that chooses to re-derive them from the authored scalars must derive them the same way, or the
same file will produce a different effect.

## `Domain(kind, a0 … a8)`

**Contract** — builds a domain from nine scalars in the order an author supplies them, filling
the positional fields and deriving everything cached. Pure; no allocation; no failure path —
an unrecognized kind leaves the record untouched rather than reporting anything.

```text
FUNCTION make_domain(kind, a0..a8) -> Domain
  d.kind = kind
  SWITCH kind
    Point      : d.p1 = (a0,a1,a2)
    Line       : d.p1 = (a0,a1,a2); d.p2 = (a3,a4,a5) - d.p1      # stored as a delta
    Box        : per axis, d.p1 = min(authored pair), d.p2 = max
    Triangle   : d.p1 = (a0,a1,a2)
                 d.u  = (a3,a4,a5) - d.p1 ; d.v = (a6,a7,a8) - d.p1
                 d.radius1Sqr = |u| ; d.radius2Sqr = |v|          # edge LENGTHS, not squares
                 d.p2 = normalize( (u/|u|) cross (v/|v|) )
                 d.radius1 = -(d.p1 . d.p2)
    Rectangle  : as Triangle, except u and v are authored directly as edge vectors
    Plane      : d.p1 = (a0,a1,a2); d.p2 = normalize(a3,a4,a5)
                 d.radius1 = -(d.p1 . d.p2)
    Sphere     : d.p1 = (a0,a1,a2)
                 d.radius1 = max(a3,a4); d.radius2 = min(a3,a4)
                 squares cached
    Cylinder,
    Cone       : d.p1 = (a0,a1,a2); d.p2 = (a3,a4,a5) - d.p1      # axis as a delta
                 d.radius1 = max(a6,a7); d.radius2 = min(a6,a7)
                 d.radius1Sqr = radius1²
                 d.radius2Sqr = 1/(p2 . p2), or 0 for a degenerate axis
                 d.u, d.v = orthonormal_frame(normalize(p2))
    Blob       : d.p1 = (a0,a1,a2); d.radius1 = a3 (sigma)
                 d.radius2Sqr = -0.5/sigma² ; d.radius2 = 1/(sigma·sqrt(2·pi))
    Disc       : d.p1 = (a0,a1,a2); d.p2 = normalize(a3,a4,a5)
                 d.radius1 = max(a6,a7); d.radius2 = min(a6,a7)
                 d.u, d.v = orthonormal_frame(d.p2)
                 d.radius1Sqr = -(d.p1 . d.p2)                    # the plane offset
  RETURN d

FUNCTION orthonormal_frame(n) -> (u, v)
  # A fixed, deterministic choice: start from the x axis unless the axis is nearly
  # parallel to it, in which case start from y. The 0.999 threshold only has to be
  # far enough from 1 that the projection below does not lose all its precision.
  seed = (1,0,0)
  IF |seed . n| > 0.999 THEN seed = (0,1,0)
  u = normalize(seed - n · (seed . n))
  v = n cross u
  RETURN (u, v)
```

## `Domain.within(point)`

**Contract** — is the point inside? Pure except for the blob, which consumes a random number
and is therefore *not* a predicate at all but a stochastic one. Returns false for every kind
that has no interior — point, line, triangle, rectangle and disc are all measure-zero regions,
and the module does not attempt a thickness.

**Invariants** — the sphere, cylinder and cone tests are shell tests: a point inside the inner
radius is *outside* the domain.

```text
FUNCTION within(d, p) -> bool
  SWITCH d.kind
    Box    : RETURN p componentwise inside [d.p1, d.p2]
    Plane  : RETURN (p . d.p2) + d.radius1 >= 0        # the positive half-space is inside
    Sphere : r2 = |p - d.p1|²
             RETURN r2 <= d.radius1Sqr AND r2 >= d.radius2Sqr
    Cylinder, Cone :
             x    = p - d.p1
             t    = (d.p2 . x) · d.radius2Sqr          # 0 at the base, 1 at the far end
             IF t < 0 OR t > 1 THEN RETURN false
             radial = x - d.p2 · t
             IF d.kind = Cone THEN
               # the radii scale linearly with distance from the apex
               RETURN |radial|² <= (t·d.radius1)² AND |radial|² >= (t·d.radius2)²
             RETURN |radial|² <= d.radius1Sqr AND |radial|² >= d.radius2²
    Blob   : # membership with probability equal to the normalized gaussian at p
             g = exp(|p - d.p1|² · d.radius2Sqr) · d.radius2
             RETURN uniform_random() < g
    ELSE   : RETURN false
```

**Notes** — the false-for-everything-else branch is load-bearing in an unobvious way. The kill
actions phrase themselves as "remove when membership equals the configured side", so an author
who points a kill action at a disc or a line gets *every particle removed* when the action is
configured to kill outsiders — not a no-op. A rebuild must reproduce the false, not
substitute a plausible proximity test.

The blob is the only domain whose containment is random. A kill action on a blob domain thins
the population probabilistically per step rather than clipping it, which is the effect an
author using it is after.

## `Domain.generate()`

**Contract** — draws one point. Consumes between one and three uniform random numbers from the
shared generator (the blob consumes an unbounded number; see `normal_random`). No failure
path: an unrecognized kind yields the origin.

**Invariants** — the distributions below are the observable contract of the module. They are
not all uniform-by-volume, and where they are not, the deviation is the authored look.

```text
FUNCTION generate(d) -> vec3
  SWITCH d.kind
    Point     : RETURN d.p1
    Line      : RETURN d.p1 + d.p2 · rand()                      # uniform along the segment
    Box       : RETURN componentwise lerp from d.p1 to d.p2 with three independent rand()
    Triangle  : r1, r2 = rand(), rand()
                # fold the far half of the unit square back over the diagonal:
                # uniform over the triangle, two random numbers, no rejection loop
                IF r1 + r2 < 1 THEN RETURN d.p1 + u·r1 + v·r2
                ELSE               RETURN d.p1 + u·(1-r1) + v·(1-r2)
    Rectangle : RETURN d.p1 + u·rand() + v·rand()
    Plane     : RETURN d.p1                                      # no sane sample exists
    Sphere    : dir = normalize( (rand(),rand(),rand()) - (0.5,0.5,0.5) )
                IF d.radius1 = d.radius2 THEN RETURN d.p1 + dir·radius1
                RETURN d.p1 + dir · (radius2 + rand()·(radius1 - radius2))
    Cylinder,
    Cone      : t     = rand()                                   # along the axis
                theta = rand() · 2·pi
                r     = radius2 + rand()·(radius1 - radius2)
                x = r·cos(theta); y = r·sin(theta)
                IF d.kind = Cone THEN x = x·t ; y = y·t          # taper toward the apex
                RETURN d.p1 + d.p2·t + u·x + v·y
    Blob      : RETURN d.p1 + (normal_random(sigma) on each axis independently)
    Disc      : theta = rand() · 2·pi
                r     = radius2 + rand()·(radius1 - radius2)
                RETURN d.p1 + u·r·cos(theta) + v·r·sin(theta)
    ELSE      : RETURN origin
```

**Notes** — three of these are deliberately *not* uniform by volume, and a rebuild that
"fixes" them changes how shipped effects look:

- The sphere picks a direction by normalizing a point drawn from the unit cube. That is biased
  toward the cube's corners — the diagonals get more samples than the axes — and it can, with
  probability zero in theory and rarely in practice, normalize a near-zero vector. The radius
  is then drawn *linearly* between the inner and outer radius, which clusters samples toward
  the centre rather than distributing them by shell area. The source comment concedes the
  point and the behaviour stays.
- The cylinder and cone likewise draw the radius linearly, so a filled disc cross-section is
  denser at the middle.
- The cone's taper multiplies the radial offset by the axial parameter, so `p1` is the apex and
  the far end is the wide end — the opposite of the "base point, height vector" reading the
  field names suggest.

The triangle's fold is the standard two-uniform trick and needs no rejection loop; preserve it
for the sake of the random-number *consumption count*, which affects every later draw when a
single shared generator feeds the whole simulation.

## `Domain.transform(source, placement)` and `transform_direction(source, placement)`

**Contract** — recomputes this domain as `source` moved by a placement matrix. The source and
destination are separate records — actions keep the authored domain untouched in local space
and re-derive the world-space copy every time the emitter moves. Never in place.
`transform_direction` is the same operation with the translation column zeroed, for domains
that hold velocities or accelerations, which must rotate with the emitter but not follow it.

An unrecognized kind is a programming error and aborts; it never silently passes through.

```text
FUNCTION transform(dst, src, m)
  SWITCH dst.kind
    Box    : transform the corner pair as an axis-aligned box — that is, transform all
             eight corners and take the extents, not the two corners, or a rotated
             placement produces an inside-out box
    Plane  : dst.p1 = m applied to src.p1 as a point
             dst.p2 = m applied to src.p2 as a direction
             dst.radius1 = -(dst.p1 . dst.p2)          # the offset must be re-derived
    Sphere,
    Point,
    Blob   : dst.p1 = m applied to src.p1 as a point
    Line   : point for p1, direction for p2
    Cylinder, Cone, Rectangle, Triangle, Disc :
             point for p1; direction for p2, u and v
    ELSE   : FAIL WITH unreachable
```

**Notes** — the radii and the cached reciprocals are *not* rescaled. A placement matrix with
non-unit scale moves a sphere's centre but leaves its radius alone, and moves a triangle's
vertices while leaving its cached edge lengths stale. Emitter placements in this engine are
rigid, so the case does not arise; a rebuild that allows scaled placements must decide what to
do here, because the original does nothing.

The disc is the one kind whose plane offset lives in `radius1Sqr`, and the transform does not
re-derive it. A disc that is moved after loading therefore keeps the offset of its authored
position, which is a latent defect in the avoid and bounce actions' disc paths; in shipped
data, discs are authored in the emitter's local space at the origin, where the stale offset
happens to be harmless.

## `normal_random(sigma)`

**Contract** — one sample from a zero-mean normal distribution with the given standard
deviation. Returns zero exactly when sigma is zero. Blocks for an unbounded but almost surely
short time: it is a rejection sampler.

```text
FUNCTION normal_random(sigma) -> real
  IF sigma = 0 THEN RETURN 0
  REPEAT
    y = -log(uniform_random())          # an exponential sample
  UNTIL uniform_random() <= exp(-((y-1)²)/2)
  # The rejection loop yields the magnitude; the sign comes from a coin flip.
  IF coin_flip() THEN RETURN  y · sigma · (1/0.7975)
  ELSE               RETURN -y · sigma · (1/0.7975)
```

**Notes** — this is the textbook exponential-envelope method for a half-normal, mirrored to
both sides. The constant `0.7975` is the mean of the magnitude the loop produces; dividing by
it makes the result's standard deviation come out at `sigma` as promised. It is a fitted
constant, not an exact one.

The sign comes from a *different* random generator than the magnitude: the magnitude is drawn
from the engine's shared uniform source, the sign from the platform's own integer generator.
That split is incidental in origin and load-bearing in consequence — reproducing this module's
exact particle stream requires both sequences, and the noise table's initializer reseeds the
second one (see [`noise.cpp`](noise.cpp.md)). A rebuild that wants determinism should draw
both from one explicitly seeded generator and accept that the particle stream will differ from
the original's.
