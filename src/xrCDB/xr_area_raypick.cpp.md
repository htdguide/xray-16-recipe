# src/xrCDB/xr_area_raypick.cpp

> The three ray questions the game actually asks — is anything in the way, what is
> the nearest thing in the way, and what is everything in the way — each merging the static
> world with the moving objects.

**Needs** — [`xr_area.h`](xr_area.h.md) · [`ISpatial.h`](ISpatial.h.md) · [`Intersect.hpp`](Intersect.hpp.md) · [`xr_collide_defs.h`](xr_collide_defs.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it orchestrates two queries and merges their results; the low-level
work is elsewhere.

## Purpose

Everything below this file knows about one world at a time: the tree knows triangles, the
octree knows objects. Every caller above it wants *one* answer about the world as a whole —
a bullet does not care whether it hit a wall or a crate. This file is that join, and the
three entry points exist because the three questions have genuinely different cost.

## State

Stateless; uses the thread's scratch from [`xr_area.cpp`](xr_area.cpp.md).

## `ObjectSpace.ray_test`

**Contract** — *is anything of the requested kinds within range along this ray?* Returns a
boolean and nothing else. Takes an optional per-caller ray cache. Clears the thread's
candidate list before returning. Requires a unit direction.

```text
FUNCTION ray_test(origin, dir, range, targets, cache, ignore) -> bool
  # dynamic first -- cheap, and most occlusion in a populated scene is an object
  IF targets include any moving kind
    candidates <- octree.ray(origin, dir, range, mask from targets)
    FOR EACH candidate that is a game object, is not `ignore`, and is of a requested kind
      IF its collision form is hit by the ray: RETURN true

  IF targets include the static world
    IF cache is given
      IF cache matches this exact ray: RETURN cache.blocked          # 0 tree work
      IF the ray hits cache's remembered triangle within range: RETURN true
      hit <- tree.ray_query(first-only)
      cache <- (this ray, whether hit, the blocking triangle's vertices)
      RETURN hit
    ELSE
      RETURN tree.ray_query(first-only) found anything
  RETURN false
```

**Three decisions here, all about cost.**

*Dynamic before static.* The reverse order would be wrong for this question: the static test
is the expensive one, and in a scene full of creatures and crates an occlusion query is more
often blocked by an object than by a wall.

*First-only, not nearest.* The question is existence, so the traversal may stop at whatever
it finds first and never shrink its range. This is the cheapest of the three result
policies, and it is the reason this entry point exists separately from `ray_pick` at all.

*The cache is a two-level memo.* An exact repeat is answered with no work; a near-repeat is
answered by testing the single triangle that blocked last time. The AI issues these from a
nearly fixed eye position every frame, so both levels hit often. The cache is owned by the
asking entity (see [`xr_collide_defs.h`](xr_collide_defs.h.md)), which is what keeps its hit
rate high — one cache per (observer, target) pair rather than one for the world.

**The cache is only updated on the tree path**, so a query blocked by a moving object
returns early and leaves the cache describing an older state. That is safe — the cache is
only ever consulted for the static test — but it means the cache does not memoize the
question the caller asked, only the static half of it.

## `ObjectSpace.ray_pick`

**Contract** — *what is the nearest thing in the way?* Fills a single hit record with the
nearest intersection, or leaves its element negative if there was none. Returns whether
anything was hit. Clears the thread's candidate list before returning.

```text
FUNCTION ray_pick(origin, dir, range, targets, out, ignore) -> bool
  out <- (no object, range, element = -1)

  IF targets include the static world
    tree.ray_query(nearest, culling)
    IF anything: out.take_if_nearer(that hit)

  IF targets include any moving kind
    candidates <- octree.ray(origin, dir, range, mask from targets)
    FOR EACH candidate that is a game object, is not `ignore`, and is of a requested kind
      test its collision form against the ray limited to out.range   # shrinking bound
      IF it hit: out.take_if_nearer(that hit)

  RETURN out.element >= 0
```

**Static first, and the order is the optimization.** The tree's nearest-only traversal
shrinks its own range as it goes and is the tighter of the two searches; running it first
gives the object loop a range bound that is usually already close to final, so most objects
are rejected by their own bounding test. Reversing the order would work and would be slower.

**The object loop re-limits the ray for every object** rather than once at the start —
each test uses the best range found so far, so a near object prunes the ones behind it.

**Culling is on for the static world.** This is the bullet-and-camera question and a surface
seen from behind does not stop it.

## `ObjectSpace.ray_query`

**Contract** — *everything in the way, nearest first, with the caller deciding when to
stop.* Fills a hit set, invoking a caller-supplied step per hit in order; the step returns
whether to continue. Takes a second caller-supplied step that vets each candidate object
before it is tested. Returns how many hits were delivered.

```text
FUNCTION ray_query(out, request, on_hit, should_test, ignore) -> int
  clear out, scratch

  IF request.target includes the static world
    tree.ray_query(request.flags)                  # whatever policy the caller asked for
    FOR EACH result: scratch.append(no object, its range, its triangle index)

  IF request.target includes any moving kind
    candidates <- octree.ray(request)
    FOR EACH candidate that is a game object, is not `ignore`, and is of a requested kind
      IF should_test is given AND it says no: CONTINUE
      test its collision form; append its hits to scratch

  IF scratch is non-empty
    scratch.sort by range
    FOR EACH hit in order
      out.append(hit)
      IF on_hit is given AND it says stop: RETURN count(out)
      IF request.flags ask for first-only or nearest-only: RETURN count(out)
  RETURN count(out)
```

**Collect everything, then sort, then deliver in order.** The two sources produce hits
independently and neither is ordered against the other, so a global sort is the only way to
deliver "nearest first" across both. The cost is that a caller who only wants the first two
hits still pays for every hit in range — which is why `ray_test` and `ray_pick` exist rather
than being thin wrappers over this.

**The per-hit callback is how penetration is expressed.** A bullet asks for everything in
the way and walks the hits outward, subtracting armour from its energy at each, stopping
when it runs out. The database does not know what stops a bullet; the callback does. This is
the only entry point that can express that, and it is the reason the general query is not
simply "nearest".

**The candidate-vetting callback lets a caller skip whole categories** — a creature's own
body parts, a friendly it will not shoot through — without the database learning about
categories.

## Notes

Two *further* ray-query strategies sit in this file unused, and they are worth naming
because they are the alternative designs and a rebuilder will reinvent one of them.

Both replace "collect everything and sort" with an **advance-and-repeat** walk: ask each
source for its nearest hit, deliver whichever is nearer, then move that source's ray origin
just past what it found and ask again, until nothing is left or the callback stops. That
delivers hits in order without ever materializing the full set, so a caller who stops after
two hits pays for two, not for all of them. The cost is one tree descent per hit instead of
one per query, plus a nudge of a small epsilon past each hit so the same triangle is not
found twice — and it is that nudge, applied to a floating-point range that has already been
reduced several times, that is delicate: the instrumented build checks at three separate
points that the accumulated range has not gone negative, which tells you it did.

The version that ships is the simple one. A rebuild should keep it and reach for the
advance-and-repeat shape only if profiling shows the unbounded collect dominating —
remembering that it changes the meaning of the range parameter from "along the original ray"
to "accumulated across segments", which is exactly where the original's arithmetic gets
fragile.

There is also a one-target form of the query that tests a single named collision form and
skips both indexes entirely, for a caller who already knows what it wants to test against.
