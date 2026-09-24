# src/xrEngine/Feel_Vision.cpp

> Decides what an entity can see: a frustum query for candidates, a set difference against last frame, and one cached, transparency-aware ray per candidate.

**Needs** — [`Feel_Vision.h`](Feel_Vision.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`xr_object.h`](xr_object.h.md) · [`xr_collide_form.h`](xr_collide_form.h.md) · [`xrCDB`](../xrCDB/README.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: spatial queries and ray casts. The triangle cache indexes the collision database's own arrays, which is the only low-level touch.

## Purpose

Sight is the most expensive sense and the one the AI leans on hardest — a few dozen creatures each checking a few dozen targets, every sense tick. Almost everything in this file exists to make that affordable without making it wrong: a broad-phase frustum query, a set difference so that only changes cost anything, a per-target aim point that moves when the current one is blocked, and a two-level ray cache.

The output is not a yes-or-no. It is a signed confidence per target and, for each visible target, *where* on it the viewer can see — which is what a creature aims at.

## `query`

**Contract** — Gathers this frame's candidate set: every entity inside a frustum built from a full transform, filtered to those the game marks as relevant to this sense. Sorted and deduplicated, because the set difference in `update` needs both. Does not trace anything and does not allocate beyond the two reused buffers.

```text
FUNCTION query(full_transform, viewpoint)
  frustum = from the transform, using the four side planes and the far plane
  candidates = spatial query for entities marked "visible to AI" inside the frustum
  seen = every candidate the game calls relevant
  sort and deduplicate seen
```

**Notes** — The near plane is deliberately excluded from the frustum. A target overlapping the viewer must remain a candidate, and a near plane would cull it.

**Notes** — The spatial query is against a *separate* category from rendering visibility. An entity can be visible to the renderer and not to the AI, and the reverse — a creature in a dark room is renderable and, without a light, is still a sight candidate because this sense models light through materials rather than through the renderer's lighting.

**Notes** — Deduplication is needed because a spatially large entity can be returned once per spatial cell it occupies.

## `update`

**Contract** — Reconciles this frame's candidate set against the previous one, creating a tracking item for each newly appearing target and dropping the item for each that has left, then traces every tracked target. The viewer's owning entity is removed from its own candidate set first.

```text
FUNCTION update(owner, viewpoint, dt, visibility_threshold)
  remove owner from seen
  newly_appeared = seen - previous                 # both sets are sorted
  FOR EACH e IN newly_appeared: create tracking item for e
  disappeared    = previous - seen
  FOR EACH e IN disappeared:   drop tracking item for e
  previous = seen
  trace(viewpoint, dt, visibility_threshold)
```

**Notes** — Two set differences over sorted sets rather than a membership test per element. With a few dozen candidates that is not a performance argument; it is a correctness one — it produces exactly the appear and disappear edges, once each, which a per-element test would have to be careful to do.

## Creating a tracking item

**Contract** — A new item starts at a confidence just below zero — not seen, but only barely — so that a single successful trace flips it positive immediately, while a target that is never traceable stays negative. It also picks a random aim point on the target's mesh, recorded in the target's own frame along with the bone it belongs to, and resolves it to a world point.

**Notes** — The aim point is *on the mesh and attached to a bone*, not the target's origin. That is the whole reason a creature can see a head over a wall while the body is hidden, and it is why the aim point is stored locally and re-resolved every trace: the bone moves with the animation.

## `trace`

**Contract** — For every tracked target, casts one ray from the viewpoint to the target's aim point through the static world and through other entities, accumulating transparency, and moves that target's confidence up or down. Trivially visible for a target closer than a guaranteed distance or exactly at the viewpoint. Updates each item's last-visible world point.

```text
FUNCTION trace(viewpoint, dt, threshold)
  FOR EACH item IN tracked
    IF the target has no collision shape THEN item.fuzzy = -1 ; CONTINUE

    item.last_visible = resolve(item.aim_point_local, item.aim_bone)   # world space
    direction = item.last_visible - viewpoint
    IF direction has zero length THEN item.fuzzy = 1 ; CONTINUE

    # A fifth of a metre is added so the ray ends just past the aim point
    # rather than exactly on the surface it is aiming at.
    range = length(direction) + 0.2
    IF range <= guaranteed_distance THEN gain confidence ; CONTINUE
    direction = normalize(direction)

    visibility = trace_static(viewpoint, direction, range, item, threshold)
    IF any other entity's collision shape blocks the ray THEN visibility = 0

    IF visibility < threshold THEN
      item.fuzzy = clamp(item.fuzzy - loss_rate * dt, -0.5, 1)
      item.aim_point_local = a NEW random point on the target's mesh   # try elsewhere next time
    ELSE
      item.fuzzy = clamp(item.fuzzy + gain_rate * dt, -0.5, 1)
```

**Invariants** — Confidence never falls below −0.5 and never rises above 1. The asymmetric floor is deliberate: a target that was seen and is now hidden decays to "half-forgotten", not to "never seen", so the game's search behaviour can tell the two apart.

**Notes** — Choosing a *new* random aim point whenever the trace fails is the core trick. One ray per target per tick cannot prove a target is invisible — it only proves one point on it is — so the sense wanders the aim point across the mesh, and over a few ticks a target that is partly exposed is found. It also means visibility is stochastic: two identical viewers can disagree for a tick.

**Notes** — The extra 0.2 on the ray length prevents the ray terminating exactly at the target's own surface, where a floating-point coincidence decides whether the target's own geometry counts as a blocker.

## The static-world trace and its cache

**Contract** — Casts a ray through the static collision database, multiplying the accumulated visibility by each hit surface's transparency as reported by the game, and stopping as soon as the accumulated value falls to or below the threshold. Caches enough to skip the cast entirely on most subsequent frames.

```text
FUNCTION trace_static(from, dir, range, item, threshold) -> real
  IF item.cache is valid AND its ray is similar to this one THEN
    RETURN item.cached_visibility                     # level 1: same ray as last time

  IF the cached blocking triangle is still hit by this ray within range THEN
    RETURN 0                                          # level 2: same wall, different ray

  visibility = 1
  cast the ray, and for EACH surface hit, in any order:
    visibility = visibility * material_transparency(hit)
    IF the hit was static geometry and its transparency is zero THEN
      remember that triangle's three vertices as the cache
    CONTINUE while visibility > threshold
  record (from, dir, range, whether anything was hit) in the cache
  item.cached_visibility = visibility
  RETURN visibility
```

**Notes** — Two cache levels, and the second is the valuable one. The first, "the same ray as last time", almost never hits in practice because both endpoints move. The second — "test only the single triangle that blocked me last time" — hits constantly, because a creature hidden behind a wall stays behind the *same* wall for hundreds of ticks, and one triangle test is dramatically cheaper than a tree descent.

**Notes** — The cached triangle is stored as three *vertices copied out of the collision database*, not as an index. That makes the cache independent of the database's lifetime, which matters because the database is rebuilt on a level change.

**Notes** — Transparencies multiply, so two panes of half-transparent glass yield a quarter. The traversal stops as soon as the product drops below the threshold, which is why the callback returns "keep going" rather than "hit" — the ray query's early-out is being used as the accumulation's termination condition.

**Notes** — When the cast finds nothing at all, the cache records the ray but marks it as not blocking, and the accumulated visibility is *not* forced to zero — the original has the zeroing commented out. The effect is that a ray that escapes the world entirely reports fully visible, which is correct.

**Notes** — Dynamic entities are traced in a second pass, and any hit at all makes the target invisible regardless of the entity's material. The viewer itself and the target itself are skipped. That asymmetry — static geometry gets graded transparency, entities get a hard block — is a simplification, not a principle: an entity's collision shape has no material to ask about at this level.

## `clear` / `on_object_released`

**Contract** — Clear forgets every set and every tracked item. The release-case handler removes one entity from all four lists, which is the engine-wide guarantee that a destroyed entity is unreferenced before its memory goes.
