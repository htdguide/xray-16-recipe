# src/xrGame/ik_foot_collider.cpp

> Finds the surface under a foot by casting three rays — toe, heel and outer edge — and reduces them to a single plane the foot will be aligned to.

**Needs** — [`ik_foot_collider.h`](ik_foot_collider.h.md) · [`ik_collide_data.h`](ik_collide_data.h.md) · [`Level.h`](Level.h.md) · [`GameObject.h`](GameObject.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrCDB/Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [Seam: Collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`ik_foot_collider.h`](ik_foot_collider.h.md)
**Tier floor** — T1: walks the collision database's triangle arrays and reads material flags per triangle

## Purpose

Before a foot can be placed, the engine must know what is under it — and "what is under it"
is not one answer. A foot on a step has its toe on one surface and its heel on another; a
foot on a kerb has its outer edge on a third. This file resolves that into a **single plane**,
which is what the solver can actually align a foot to, and it does so by a rule that
prioritizes not falling through the world over looking correct.

Three things here are load-bearing beyond the geometry: the material filter (some surfaces
are not floors), the penetration loop (a ray must pass *through* non-floor surfaces rather
than stopping at them), and the caching (three ray casts per foot per frame is expensive, and
a standing character's queries do not change).

## State

```text
RECORD FootCollider
  previous_toe, previous_heel, previous_side : PickQuery
  previous_result : CollideResult
```

## What counts as ground

**Contract** — a surface is ignored, and the ray continues through it, when either:

- its material is marked passable **and not** marked an actor obstacle — a bush, a curtain,
  a low fence the player walks through; or
- its material is marked climbable — a ladder.

A surface belonging to a **living entity** is always ignored. Feet do not stand on creatures.

**Invariants** — the passable test is a conjunction, not a single flag: a material can be
passable to projectiles and still an obstacle to an actor, and that combination must remain
ground. Dropping the second half of the test makes characters sink through every fence in the
game.

Ladders are ignored despite being solid, because a foot aligned to a ladder rung's face is
aligned to a vertical plane.

**Notes** — the living-entity test deliberately does **not** check whether the entity is
alive; the check is commented out in the source. So a *corpse* is also not ground, and a
character walking over a body has their feet pass through it. That is the shipped behaviour.

## Reading a surface

**Contract** — a hit is turned into a plane, a contact position, a triangle and a distance.
The two cases differ in where the geometry comes from but agree on the result.

For **static level geometry**: the triangle's three vertices are read from the collision
database, and the plane is built from them with the normal **inverted**.

**Invariants** — the inversion is the frozen convention in this file: a ground plane's normal
points *along* the pick direction, that is, downward. Every dot product below assumes it. A
rebuild that keeps the outward normal must flip every comparison.

For **dynamic objects**: the hit gives an object and a bone, and the object's skinned geometry
is re-queried at that bone to get the precise triangle, because the broad-phase result is
against a bounding volume rather than against the deformed mesh. If the object has no skinned
geometry the hit is discarded.

## The penetration loop

**Contract** — casting one ray is not enough, because the first thing it hits may be a
surface that is not ground. The query walks forward through such surfaces until it finds one
that is.

```text
FUNCTION pick(query, object_to_ignore) -> optional<Surface>
  position = query.position
  range    = query.range
  WHILE the collision database finds a hit along (position, query.direction, range)
    IF the hit is real ground THEN RETURN its surface

    # step past the surface we just declined and continue
    position = the hit point, advanced by a small epsilon along the ray
    IF the ray is entering the surface from behind (direction agrees with its normal) THEN
      advance the position a further epsilon ALONG THE SURFACE NORMAL
    range = range - distance consumed - epsilon
    IF range < epsilon THEN BREAK
  RETURN nothing
```

**Invariants** — the extra nudge along the normal when the ray and the surface normal agree
is what prevents an infinite loop on a surface the ray is grazing: advancing only along the
ray direction can land the new origin back on the same triangle, and the query finds it again
at zero distance. The condition tests the *sign of the agreement*, which is the case where
stepping along the ray alone does not escape.

The remaining range is decremented by the consumed distance **and** the epsilon, so the total
search distance is bounded no matter how many surfaces are penetrated.

## `collide`

**Contract** — runs the ground test for one foot and fills in the result: whether anything was
hit, the plane to align to, and which of the three points produced it.

```text
FUNCTION collide(result, foot_geometry, owner, is_foot_step)
  REQUIRE the foot geometry has been set
  result.collided = false

  search_distance = collide_dist + reach_dist          # 0.5 above + 1.5 of reach

  FOR EACH point IN (toe, heel, side)
    start = foot_geometry[point] - pick_direction * collide_dist    # start ABOVE the point
    query[point] = PickQuery(point, start, pick_direction, search_distance)

  IF all three queries equal last frame's THEN
    result = previous_result ; RETURN                  # nothing moved: reuse

  remember the three queries
  foot_length = distance(toe_start, heel_start) * 1.5  # the compatibility radius

  toe_hit  = pick(query[toe],  ignoring the owner)
  heel_hit = pick(query[heel], ignoring the owner)
  side_hit = pick(query[side], ignoring the owner)

  # --- case 1: all three on one surface ---------------------------------------
  toe_heel_ok = toe_hit AND heel_hit AND distance(heel, toe) < foot_length
  toe_side_ok = toe_hit AND side_hit AND distance(side, toe) < foot_length
  IF toe_heel_ok AND toe_side_ok THEN
    plane = the plane through the three CONTACT POINTS
    IF that plane faces away from the toe's own surface normal THEN flip it
    result.plane = plane
    remember result ; RETURN

  # --- case 2: fall back to the HIGHEST single contact ------------------------
  best = toe_hit (if any)
  IF heel_hit is higher THEN best = heel_hit
  IF side_hit is higher THEN best = side_hit
  IF any hit THEN
    result.plane = best.plane ; result.point = best.point
  remember result
```

**Invariants**, in order of how much they matter:

- **The three queries all use the same direction**, taken from the incoming result's pick
  direction. The three positions differ; the direction does not. The code computes the
  direction three times from the same source, which is redundant but documents that the three
  rays are parallel — a per-point direction would make the plane through the three contacts
  meaningless.

- **The compatibility test is a distance, not a coplanarity test.** Two contacts count as "on
  the same surface" when they are no further apart than one and a half foot lengths. That
  crude test is what rejects the case where the toe found the floor and the heel found the
  bottom of a stairwell two metres below. The factor of one and a half is slack for the fact
  that the contacts are on a slope and so further apart than the foot's flat length; it is an
  empirical figure.

- **The single-contact fallback takes the HIGHEST contact, by world height.** Not the nearest
  along the ray, not the one whose normal is most upward — the highest. This is the rule that
  keeps a foot out of the floor: aligning to the highest thing the foot touches guarantees no
  part of the foot ends up below a surface it is standing on. It also means a foot on a step
  aligns to the upper tread.

- **The three-point plane is flipped to agree with the toe's own surface normal.** The plane
  through three contacts has an arbitrary winding; the toe's surface is the reference because
  the toe is the query the result is otherwise keyed on.

- **The result is cached under the exact three queries.** A standing character's foot
  positions do not change, so the three ray casts — each of which may walk through several
  surfaces — are skipped entirely. This is the single largest cost saving in the foot-placement
  system, and it is safe only because the queries encode both the foot's position and the
  reach; a world that changed under a stationary foot would not invalidate it. The engine
  accepts that.

**Notes** — the reach distance is added unconditionally, although the source shows it was
once conditional on the animation saying this foot is planted. As shipped, a foot that the
animation says is *in the air* is still searched a full two metres for ground. The `is_foot_step`
argument is therefore unused. A rebuild that restores the condition will make airborne feet
cheaper and will change where they are placed.

The starting offset above the foot point is subtracted along the pick direction rather than
added along the vertical, so a limb with a non-vertical pick direction starts its ray offset
along that direction. That is consistent, and it is why the pick direction is part of the
per-limb state rather than a constant.

## `chose_best_plane`

**Contract** — picks whichever of three planes faces most directly into a given direction.

**Notes** — not called. It belongs to an earlier resolution rule that chose a plane by
orientation rather than by contact height, preserved in the source alongside the commented-out
version of `collide` it served. A rebuild should omit it; it is recorded here because it shows
the rule that was rejected — orientation-based selection allows a foot to be placed below a
surface, which is why height-based selection replaced it.

## `reach_dist`

```text
reach_dist = 1.5    # metres, added to the search distance below the foot
```

**Contract** — how far below a foot ground is looked for. Two metres of total search (half a
metre above the foot plus this) is how far a foot may be stretched down toward a step.

**Notes** — a bare constant with no derivation. It bounds how deep a hole a character's foot
will reach into before the limb gives up and the animation's own placement stands.
