# src/xrPhysics/tri-colliderknoopc/dSortTriPrimitive.h

> The traversal at the centre of the chapter: it asks the level what is nearby, decides whether
> the shape is inside the world or outside it, and picks the one triangle that will push it
> out.

**Needs** — [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`dTriColliderCommon.h`](dTriColliderCommon.h.md) · [`dTriColliderMath.h`](dTriColliderMath.h.md) · [`__aabb_tri.h`](__aabb_tri.h.md) · [`dTriCollideK.h`](dTriCollideK.h.md) · [`../ExtendedGeom.h`](../ExtendedGeom.h.md) · [`../PHWorld.h`](../PHWorld.h.md) · [`../console_vars.h`](../console_vars.h.md) · [`../../xrCDB/xr_area.h`](../../xrCDB/xr_area.h.md) · [`../../xrMaterialSystem/GameMtlLib.h`](../../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md)
**Tier floor** — T1: a per-step hot loop reading triangles out of a mapped level structure and
writing contacts into the solver's array.

## Purpose

Everything the engine does against static geometry passes through this one routine. Read it as
the answer to a harder question than "do these overlap": a triangle soup has no inside and no
outside, so a shape that has got behind a wall cannot be told from one that is resting on it
by geometry alone. The routine's real job is to **keep shapes on the side of the world they
started on**, and the machinery below — the swept plane crossing, the retained "negative"
triangle, the fall-back trajectory contact — all exists for that.

Three ideas carry the file, and a rebuilder who takes only these has the design:

1. **The candidate set is cached across steps.** A collision-database query is expensive; the
   query box is inflated by two steps of travel so its answer can be reused until the shape
   leaves it.
2. **Contacts and recovery are different modes.** A shape *in front of* a triangle gets a
   normal contact. A shape *behind* one gets pushed out along that triangle's plane, with no
   containment test — and only one triangle is chosen to do the pushing, because two
   disagreeing normals cancel and the shape stays stuck.
3. **The crossing is detected by sweeping the centre, not by overlap.** The shape's previous
   centre and its current one form a segment; if that segment crosses a triangle's plane
   inside the triangle, the shape passed through and must be recovered. This is what stops
   fast bodies tunnelling without any continuous collision detection.

## State

Per shape, carried across steps in the shape's own user data
([`../ExtendedGeom.h`](../ExtendedGeom.h.md)):

```text
  last_pos        : position         # the shape centre at the end of the last step,
                                     # or a sentinel meaning "no history"
  last_query_box  : box              # the region the cached candidates cover
  cached_tries    : list<int>        # indices into the level's triangle array
  pushing_neg     : bool             # we are currently being pushed out by neg_tri
  pushing_b_neg   : bool             # likewise for the passable-material one
  neg_tri         : triangle ref     # the triangle doing the pushing
  b_neg_tri       : triangle ref     # the passable-material one
```

**Invariants** — `last_pos` is updated **only when the shape is not being pushed out**.
That one line is the crux: while recovery is in progress the history must stay pinned at the
last known-good position, or each step re-bases the "where it came from" to a point already
inside the wall and the shape can never be told it is inside.

The two `pushing_*` flags are latched across steps: recovery persists until the triangle
reports the shape is in front of it again.

## the cached query

**Contract** — reuse the previous candidate set while the new query box still fits inside the
old one; otherwise re-query the collision database with the box inflated by a configurable
rate and replace the cache.

```text
IF no history OR NOT last_query_box.contains(current_box)
  inflated := current_box.extents * ph_tri_query_ex_aabb_rate
  candidates := static_collision_database.box_query(centre, inflated)
  cached_tries := the ids of every result
  last_query_box := box(centre, inflated)
```

**Notes** — the inflation rate is a console variable
([`../console_vars.h`](../console_vars.h.md)), so the query-versus-reuse balance is tunable at
run time; the debug counters for reused and new queries per step
([`../debug_output.h`](../debug_output.h.md)) exist to judge exactly this. The box handed in
was already inflated by two steps of velocity
([`dcTriListCollider.cpp`](dcTriListCollider.cpp.md)); this multiplies that again. A
containment test rather than an intersection test is what makes the reuse safe: the cache is
valid only while the *whole* new region was covered by the old query.

## the retained triangles

**Contract** — before touching the candidate list, re-evaluate the two triangles retained from
last step, if recovery was in progress.

```text
IF pushing_neg
  recompute neg_tri against the current centre
  contains := is the centre over neg_tri's face?
  IF neg_tri.distance < 0            # still behind it
     OR (NOT contains AND there is history)   # slid off the side while recovering
    depth := primitive.reach(along neg_tri.normal) - neg_tri.distance
    intersect := true                # we are inside the world; skip normal contacts
  ELSE
    pushing_neg := false             # recovered

IF pushing_b_neg                     # same, for the passable-material triangle,
  ...                                # without the containment clause
```

**Invariants** — the "slid off the side" clause has no analogue for the passable triangle, and
that asymmetry is real. A shape being pushed out through a solid wall that slides past the
edge of the pushing triangle is still inside the world and must keep being pushed; a shape
being pushed out of a *passable* volume (deep water, a bush, a trigger region) that slides off
the edge has simply left the volume.

## the traversal

**Contract** — walk every cached candidate, classify it, and accumulate two things: the
contacts for triangles the shape is in front of, and the single best triangle for pushing the
shape out if it is behind.

```text
FOR EACH candidate index I
  fetch its three vertices
  IF NOT cheap_box_overlap(centre, extents, vertices) THEN CONTINUE
  tri := prepare(candidate, centre)          # edges, normal, signed distance

  IF tri.distance < 0                        # the centre is BEHIND this triangle
    last_distance := signed distance of last_pos from tri's plane
    IF last_distance was NOT negative OR we are already being pushed
      IF full_box_overlap(centre, extents, vertices)
         ... see "classifying a negative triangle" below

  ELSE                                       # the centre is IN FRONT
    IF the contact budget is nearly spent THEN CONTINUE
    IF not pushing AND (not intersecting OR no history)
      contacts += primitive.Collide(vertices, tri, ...)   # the ordinary case
    IF no history
      remember tri in positive_tries         # needed by the veto below
```

**Invariants** — a negative triangle is only considered if the shape's *previous* centre was
in front of it, or recovery is already in progress. A triangle the shape was behind last step
and is still behind is not a crossing; it is the far side of a wall the shape is walking
alongside, and treating it as a crossing would push the shape through that wall.

The ordinary-contact path is suppressed entirely once the shape is known to be inside the
world. Mixing recovery pushes with ordinary contacts produces opposing normals in the same
step and the shape jitters in place — the single most recognisable symptom of getting this
wrong.

## classifying a negative triangle

**Contract** — for a candidate the shape is behind and that genuinely overlaps, decide whether
the shape *crossed* it this step, and whether it is a candidate for pushing.

```text
material  := material_of(tri)
passable  := material is marked PASSABLE
contains  := is the centre over tri's face?

IF not already pushing AND nothing has yet been found to have been crossed
  IF there is history AND NOT passable
    crossing := the point where the segment last_pos → centre meets tri's plane
    IF that point is inside the triangle
      intersect := true                      # the shape passed THROUGH this triangle
      mark this one as "the crossing"
  ELSE
    IF contains AND primitive.reach(tri.normal) > -tri.distance
      intersect := true                      # the shape is merely deep enough
ELSE
  intersect := true

IF NOT passable AND (this was the crossing OR (contains AND no history))
  depth := primitive.reach(tri.normal) - tri.distance
  IF depth is SHALLOWER than the best so far
     AND tri's normal is not opposed to either retained triangle's normal
    adopt tri as the new best negative triangle

IF NOT the crossing AND passable
  the same comparison, against the passable slot
```

**Invariants** — two rules here are the difference between working and not.

*The shallowest negative triangle wins, not the deepest.* The chosen triangle is the one the
shape has penetrated **least**, which is the nearest way out. Choosing the deepest pushes the
shape further into the geometry it is already inside.

*A candidate is rejected if its normal opposes a retained triangle's normal* — specifically if
the dot product is below −1/√2, i.e. more than 135° apart. A shape wedged in a corner is behind
two triangles that face each other; alternating between them pushes it back and forth forever.
Pinning the choice to the side it was already being pushed from is what lets it escape.

**Notes** — passable materials are tracked in a *parallel slot* rather than being ignored.
Water, deep snow and foliage still push a shape out — gently, with their own material's
friction — and they must never be chosen as the crossing, because passing through them is
legal. Giving them their own retained triangle and their own push is what lets a shape be
inside water and outside a wall simultaneously.

The first triangle found to have been crossed latches a flag that stops any later triangle
being considered a crossing in the same step. Only one crossing per step is meaningful: the
shape went through *something*, and the segment tells you which.

## emitting the result

**Contract** — after the walk, turn the two retained triangles into contacts.

```text
IF intersect AND a negative triangle was found
  IF no history                              # a fresh shape, possibly spawned in a wall
    veto it if some triangle the shape is IN FRONT of both contains the centre
      and "covers" the negative triangle                       # see below
  IF not vetoed
    IF the centre is over the negative triangle's face
      contacts := primitive.CollidePlain(tri's plane, ...)     # push out along the plane
      pushing_neg := (that produced contacts)
    ELSE
      contacts := one contact pointing back along last_pos → centre   # see below

IF a passable negative triangle was found
  the same veto, then CollidePlain against it, appended (or replacing, if the
  solid push produced nothing)

IF NOT pushing_neg
  last_pos := centre                         # only now is the history advanced
```

**Invariants** — the history is advanced only when recovery is not in progress, as stated
above. Everything else in this routine depends on it.

**The back-trajectory contact** is the fallback when the shape is behind a triangle but not
over its face — it went through an edge, or it has slid past. There is no sensible surface
normal, so the contact's normal is *the reverse of the direction the shape travelled*, its
depth is the distance travelled, and its position is the current centre. It carries the
triangle's material, and it fires the shape's contact callback like any other contact. In
plain terms: **send it back the way it came**. When the travel distance is degenerate the
normal falls back to straight down, which is the only direction that is always safe in a world
with gravity.

**The positive-triangle veto** guards the case with no history at all — a shape that has just
been created or teleported, where there is no "where it came from" to sweep from. For such a
shape, being behind a triangle is not evidence of anything: it may be legitimately standing on
the back of a one-sided surface. The veto asks whether any triangle the shape is in *front* of
both contains it and covers the negative one; if so, the negative one is ignored.

```text
FUNCTION covered(negative, positive, vertices) -> bool
  # the two share all three vertices (they are the same triangle, doubled), or
  # some vertex of the positive triangle that they do NOT share lies behind the
  # negative triangle's plane
```

**Notes** — the second half of `covered` is the geometric statement of "this positive triangle
is part of the same surface, or sits behind it, so the negative triangle is not a wall the
shape is inside of". The all-three-shared case catches double-sided geometry, which the level
data contains: a curtain or a fence is two coincident triangles wound opposite ways, and
without this clause a shape resting on one is forever "inside" the other.

The contact budget is checked as "the requested maximum minus ten" at two points. The slack
exists because `CollidePlain` can emit up to three contacts in one call and the caller's array
must not be overrun. Ten rather than three is unexplained margin.

## Notes

The routine is written as one function with several latched booleans and a pair of retained
slots, and it is genuinely difficult to follow. A rebuild should keep the *states* explicit —
`outside`, `crossed_this_step`, `recovering_from(triangle)`, `submerged_in(triangle)` — rather
than reconstructing them from flags. But the states themselves, and the transitions above, are
what the shipped game's movement feels like, and changing them changes how the player walks.

One consequence to be aware of: this design makes a shape's collision against the world depend
on its own history, which means it is **not a pure function of the current world state**.
Determinism ([conformance criterion 8](../../../SYSTEM-REQUIREMENTS.md#6-conformance)) is
therefore only guaranteed when that per-shape history is restored along with everything else —
a saved game or a network re-sync must carry `last_pos` and the two retained triangles, or the
same level and the same inputs can diverge.
