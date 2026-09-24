# src/xrPhysics/tri-colliderknoopc/dTriSphere.cpp

> Sphere against triangle: face, then edge, then vertex — and the rule that stops a shared
> edge producing two pushes.

**Needs** — [`dTriSphere.h`](dTriSphere.h.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`dTriColliderMath.h`](dTriColliderMath.h.md) · [`dTriColliderCommon.h`](dTriColliderCommon.h.md) · [`../ExtendedGeom.h`](../ExtendedGeom.h.md) · [`../PHWorld.h`](../PHWorld.h.md) · [`../../xrCDB/xr_area.h`](../../xrCDB/xr_area.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`dTriSphere.h`](dTriSphere.h.md)
**Tier floor** — T1: writes contacts into the solver's array and stamps the surface
parameters.

## Purpose

The simplest of the three primitive-versus-triangle cases, and therefore the one to read
first: it shows the shape that the box and cylinder cases repeat at greater length.

A sphere touching a triangle touches it in exactly one of three ways — on the face, on an
edge, or at a vertex — and the three are tried in that order, because each is cheaper and
more common than the next. The interesting part is the third decision: what to do when a
contact would come from a feature that a *neighbouring* triangle has already claimed.

## Stateless.

## `dTriSphere` — the ordinary case

**Contract** — the sphere is in front of the triangle's plane. Produce at most one contact.

```text
FUNCTION sphere_vs_triangle(v0, v1, v2, tri, sphere, mesh, contacts) -> int
  depth := sphere.radius - tri.distance
  IF depth < 0 THEN RETURN 0                    # too far from the plane

  IF the centre is over the triangle's face
    normal := tri.normal                        # a FACE contact
  ELSE
    IF any of this triangle's three edges is already claimed THEN RETURN 0
    IF the sphere reaches edge (v0,v1)  THEN claim that edge; normal from it
    ELSE IF edge (v1,v2)                THEN claim it;        normal from it
    ELSE IF edge (v2,v0)                THEN claim it;        normal from it
    ELSE
      IF any of this triangle's three vertices is already claimed THEN RETURN 0
      IF the sphere reaches v0 THEN claim v0; normal from it
      ELSE IF v1 ... ELSE IF v2 ...
      ELSE RETURN 0                             # over an edge region but not reaching

  emit one contact:
    normal   := -normal                         # library convention: toward the sphere
    depth    := depth  (rewritten by whichever test succeeded)
    position := sphere.centre - normal * radius
    first    := the mesh shape; second := the sphere
  record the triangle's material on the sphere's user data
  fire the sphere's contact callback with this triangle
  stamp the triangle's material into the contact's surface parameters
  RETURN 1
```

**Invariants** — the claim check happens **before** the geometric test and the claim itself
happens **after** it succeeds. Checking first is what makes the rule work: the second triangle
of a shared edge does not test at all, so there is no second contact to discard. See
[`dcTriListCollider.h`](dcTriListCollider.h.md) for what claiming does.

The two tiers are independent: a triangle whose edges are claimed still gets to try its
vertices, because an edge claim from one neighbour does not mean the vertex is covered.

**Notes** — the order face → edge → vertex is not arbitrary. It is the order of decreasing
likelihood *and* of decreasing quality: a face contact has a well-defined normal, an edge
contact's normal is the perpendicular from the edge (less stable), and a vertex contact's
normal is radial (least stable, and the one most likely to launch a rolling sphere). Trying
face first means the good case is also the fast case.

The material is written twice — once onto the sphere's user data as "the last triangle
material I touched", and once into the contact's surface parameters. The first is what the
game layer reads to know what surface a creature is standing on (footstep sounds, footprint
decals); the second is what the contact-tuning pass reads to set friction and bounce. Two
consumers, two places.

## the three geometric tests

**Contract** — each returns whether the sphere reaches the feature, and if so the outward
normal and the depth.

```text
point test:    normal := centre - point;  depth := radius - |normal|
               reject if the distance exceeds the radius
               if the distance is zero, the normal falls back to straight up

segment test:  project the centre onto the segment, parametrically
               reject if the projection falls outside [0,1]   # not this edge's region
               reject if the perpendicular distance exceeds the radius
               normal := from the projection to the centre;  depth := radius - that
```

**Invariants** — the segment test rejects when the projection falls outside the segment, which
is what hands the case down to the vertex tier. Clamping the projection into the segment
instead would make every edge test also a vertex test and the claim bookkeeping would stop
working.

**Notes** — both tests fall back to a straight-up normal when the distance is exactly zero —
the sphere's centre is on the feature. There is no correct answer there and up is the only
choice that does not make a creature standing on a seam sink.

The file carries two implementations of each test: an older pair returning a depth or −1, and
a newer pair returning a boolean with depth as an out-parameter. Only the newer are called.
The older ones differ in a way worth noting: the old segment test compares projections along
the *normalised* edge direction and can accept a projection slightly outside the segment, while
the new one is parametric and cannot. The switch is a bug fix, and the dead code is residue.

## `dSortedTriSphere` — the recovery case

**Contract** — the sphere is *behind* the triangle's plane and must be pushed out. No
containment test, no edge or vertex tiers, no engagement.

```text
FUNCTION sphere_vs_plane(tri_normal, triangle, distance, sphere, mesh, contacts) -> int
  depth := radius - distance
  IF depth < 0 THEN RETURN 0
  emit one contact along the triangle's plane normal, at the sphere's surface
  RETURN 1
```

**Notes** — this is deliberately the crudest routine in the file. The shape is inside the
world; the only thing that matters is getting it out along a normal that will not fight the
next step's push. The containment test is *wrong* here — the sphere is behind the triangle
precisely because it is no longer over the face — and consulting engagement would be worse,
because a neighbouring triangle having claimed an edge is no reason to leave a shape stuck
inside a wall.
