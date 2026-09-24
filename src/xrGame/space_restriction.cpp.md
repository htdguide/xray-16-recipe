# src/xrGame/space_restriction.cpp

> One entity's *effective movement space*: the permitted volumes it must stay inside, minus the forbidden volumes it must stay out of, reduced to a single navigation-mesh border that can be stamped onto the level graph as a barrier.

**Needs** — [`space_restriction.h`](space_restriction.h.md) · [`space_restriction_inline.h`](space_restriction_inline.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`space_restriction_base.h`](space_restriction_base.h.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_abstract.h`](space_restriction_abstract.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: set operations over sorted vertex lists, on the pathfinding path but not per-frame

## Purpose

A *restrictor* is one authored volume. An entity is usually restricted by several at once,
in two opposite senses, and the pathfinder cannot afford to consult them one at a time.
This file is where the several become one: it resolves an entity's two name lists into two
composed restrictions, works out the boundary of their difference, and caches that
boundary as a single sorted vertex list. Everything downstream — the accessibility tests,
the nearest-reachable-point search, the barrier the pathfinder actually searches against —
reads that one list.

The file exists separately from the manager because the *combination* is the cacheable
thing. Two different entities restricted by the same pair of lists share one of these
objects, which is why it is reference-counted rather than owned.

## The algebra

Two lists of restrictor names come in.

- **Out restrictions** — volumes the entity may not leave. It is legal to be *inside* them.
- **In restrictions** — volumes the entity may not enter. It is legal to be *outside* them.

Each list is composed into one restriction whose `inside` is the **union** of its members
(see [`space_restriction_composition.cpp`](space_restriction_composition.cpp.md)). So the
effective space is

```text
permitted = (union of the out volumes) MINUS (union of the in volumes)
```

with either side optional: no out list means the whole level; no in list means nothing is
subtracted; neither means no restriction at all. The glossary's "intersection of its
restrictors" is the intersection of the two *constraints*, not of the volumes within one
list — several out-restrictors widen the permitted space, they do not narrow it. A rebuild
that reads them as an intersection will strand every creature that is restricted to more
than one region.

## State

```text
RECORD SpaceRestriction
  out_names   : text                       # normalized, comma-joined restrictor names
  in_names    : text                       # normalized, comma-joined restrictor names
  out         : optional<restriction>      # the composed permitted volume, by handle
  in          : optional<restriction>      # the composed forbidden volume, by handle
  initialized : bool                       # border has been built (inherited)
  border      : list<vertex>               # the merged boundary (inherited)
  applied     : bool                       # border is currently stamped on the level graph
  manager     : reference to the owning manager
```

**Invariants**

- `initialized` is false until **both** named restrictions resolve *and* build their own
  borders. A restriction naming a restrictor that has not spawned yet stays uninitialized,
  and every accessibility query on it answers *accessible*. That is the deliberate failure
  mode: an unresolved restriction is inert, never total. A rebuild that treats
  "unresolved" as "forbidden" freezes creatures at level load.
- `applied` alternates strictly. The border may be stamped exactly once before it is
  cleared, and the entity's restriction record may not be swapped while it is stamped —
  otherwise the clear would remove a different mask than the one that was set, leaving a
  permanent phantom barrier on the level graph. The manager asserts this.
- `border` here is sorted by **vertex identifier** — unlike the borders produced by
  [`space_restriction_base.cpp`](space_restriction_base.cpp.md), which are sorted by packed
  horizontal position. The two orders serve different consumers and must not be conflated:
  this list is only ever handed to the level graph's mask operations, which do not care
  about order, and deduplicated, which does.

## `initialize`

**Contract** — resolve both name lists to composed restrictions through the manager, drive
each to build its border, and then merge the two borders into one. Idempotent only in the
sense that it may be re-attempted: it either completes and sets the initialized flag, or
returns having changed nothing observable, and the caller retries on the next query. No
error is raised for an unresolvable name.

```text
FUNCTION initialize()
  out = manager.restriction(out_names)     # none when the list is empty
  in  = manager.restriction(in_names)

  IF out IS none AND in IS none
    initialized = true                     # unrestricted, and legitimately so
    RETURN

  IF out EXISTS AND NOT out.initialized    out.initialize()
  IF in  EXISTS AND NOT in.initialized     in.initialize()

  IF (out EXISTS AND NOT out.initialized) OR (in EXISTS AND NOT in.initialized)
    RETURN                                 # a member is not ready; stay inert, retry later

  IF out EXISTS
    merge_in_out_borders()
  ELSE
    border = in.border                     # subtracting from the whole level

  initialized = true
```

**Notes** — the retry is driven from the query side: every accessibility entry point
begins by calling `initialize` if the flag is clear and answering *accessible* if it is
still clear afterwards. There is no retry queue and no event; the restriction simply
becomes real the first time it is asked a question after its restrictors exist.

In checked builds the out-side composition is asked whether its border passed the
connectivity self-test, and a failure is reported by name. That is where a leaky authored
restrictor is caught.

## `merge_in_out_restrictions` — the boundary of a difference

**Contract** — produce the boundary of `out MINUS in` as a sorted, deduplicated vertex
list, from the two members' own boundaries. Pure with respect to the members; writes only
this object's border. Allocates a scratch copy of the in-side border.

```text
FUNCTION merge_in_out_borders()
  border = copy of out.border
  # Drop the stretches of the permitted volume's rim that are buried inside a
  # forbidden volume: there the real boundary is the forbidden volume's rim, not this one.
  REMOVE v FROM border WHERE in EXISTS AND in.inside(v, partially = false)

  IF in EXISTS
    temp = copy of in.border
    # Drop the stretches of the forbidden volume's rim that lie wholly outside the
    # permitted volume: those are not boundaries of the difference, they are outside it.
    REMOVE v FROM temp WHERE NOT out.inside(v, partially = true)
    APPEND temp TO border

  SORT border ; REMOVE duplicates
```

**Invariants** — the two strictness flags are opposite on purpose and neither may be
relaxed. A permitted-rim vertex is discarded only when it is **fully** inside the forbidden
volume, so a vertex that merely clips a forbidden volume stays a boundary. A forbidden-rim
vertex is kept when it is **partially** inside the permitted volume, so a vertex that
merely clips the permitted region still walls it off. Both choices err toward keeping the
barrier, because a missing border vertex is a hole a creature walks through and a spurious
one only costs a slightly smaller reachable region.

**Notes** — when either member is absent the removal predicate keeps everything, which is
how the both-present case and the single-member case share one routine.

## `accessible` (sphere)

**Contract** — can a body of the given radius stand at the given point? Initializes on
demand; answers *true* for an uninitialized restriction. Reads the level graph.

```text
FUNCTION accessible(sphere) -> bool
  IF NOT initialized
    initialize()
    IF NOT initialized  RETURN true

  IF sphere.centre is not over a valid navigation vertex  RETURN false

  IF out EXISTS
    IF NOT out.inside(sphere)              RETURN false
    IF out.on_border(sphere.centre)        RETURN false
    IF out.out_of_border(sphere.centre)    RETURN false

  IF in EXISTS
    IF in.inside(sphere)                   RETURN false
    IF in.on_border(sphere.centre)         RETURN false

  RETURN true
```

**Invariants** — a position *on* the border is never accessible, on either side. The border
is the barrier the pathfinder searches against; letting a body stand on it would put the
body in the wall. The extra `out_of_border` test on the permitted side catches the case
where the point is inside the volume by the sphere test but its own navigation vertex is
not — a point hanging over a ledge inside the volume, say — which the in-side does not need
because being outside a forbidden volume is already the safe answer.

## `accessible` (vertex, radius)

**Contract** — the same question asked about a whole navigation cell rather than a point.
Initializes on demand; answers *true* when uninitialized.

```text
FUNCTION accessible(vertex, radius) -> bool
  RETURN (out IS none OR out.inside(vertex, partially = false, radius))
     AND (in  IS none OR NOT in.inside(vertex, partially = true, radius))
```

**Invariants** — the strictness flags invert between the two sides, and that inversion is
the whole content of the function. A cell is accessible only if it is **entirely** within
the permitted volume and **not even partially** within a forbidden one. Both readings are
the conservative one; a rebuild that uses the same flag on both sides will let creatures
clip into restricted geometry on one side and refuse legal cells on the other.

## `accessible_nearest`

**Contract** — given a position that may be illegal, return the nearest legal navigation
vertex and a legal point within it. Never blocks; hard-fails if neither member exists.

```text
FUNCTION accessible_nearest(position) -> (vertex, point)
  IF out EXISTS
    # Search against THIS restriction's combined border and combined inside test,
    # so the answer respects the subtraction, not just the permitted volume.
    RETURN out.accessible_nearest(using = self, position, out_restriction = true)

  REQUIRE in EXISTS
  # With nothing to be inside of, the forbidden volume answers for itself.
  RETURN in.accessible_nearest(using = in, position, out_restriction = false)
```

**Notes** — the two branches differ in *which object's* border and inside-test the search
walks, not in the search. The search itself is one routine, in
[`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md). Passing `self`
in the first branch is what makes the result respect both constraints at once; a rebuild
that searches the permitted volume alone will happily return a point inside a forbidden
one.

## `affect`

**Contract** — four overloads asking whether a given restriction is *relevant* to a
position, a vertex with a radius, a movement between two positions, or a movement between
two vertices. Relevance means "not already inside it". The pair-of-endpoints forms answer
true when either endpoint is affected.

**Notes** — this is vestigial in the shipped build. It exists to serve an alternative
policy in which forbidden volumes are enabled lazily, one at a time, as an entity
approaches them, instead of all being merged up front; that policy is compiled out (see
below) and nothing else calls these. The commented-out body shows the intended test — a
proximity check against the nearest reachable point, with a fixed lookahead distance —
which was replaced by the far cheaper "is it inside" and never restored. A rebuild can drop
these entirely.

## `name`

**Contract** — the restriction's identity is its out-restriction list. Two restrictions
differing only in their forbidden volumes report the same name, which is fine because this
name is used for diagnostics, not as a cache key; the manager keys on both lists.

## `merge` and the lazily-enabled forbidden volumes

**Contract** — `merge` builds a composite restriction from a restriction plus a set of
others by joining their names with commas and asking the manager to resolve the result,
relying on the manager's own normalization and cache to return one shared object.
`merge_free_in_retrictions` groups an entity's forbidden volumes into connected clusters —
repeatedly finding two whose borders touch and replacing them with their merged composite,
until none touch — so that each cluster can be stamped onto the level graph only when the
entity is near it.

**Notes** — the whole path is **compiled out** in the shipped engine, and its absence is a
performance decision with a behavioural cost: with it off, every forbidden volume's border
is stamped at once, so an entity restricted by many zones pays for all of them on every
search. With it on, only the clusters the entity is actually approaching are stamped. It
was disabled rather than deleted, which suggests it was correct but not worth its
complexity. A rebuild should implement the simple path and treat the clustered one as an
optimization to reach for only if border stamping shows up in a profile.

The touching test it depends on asks whether two borders share a vertex or whether either
contains a vertex of the other, and finishes with a sorted-set intersection. That last step
assumes both lists are sorted by vertex identifier, while borders built by the base are
sorted by horizontal position — so the clustered path would need that reconciled before it
could be switched back on. This is the kind of latent inconsistency that only stays latent
because the code is dead.
