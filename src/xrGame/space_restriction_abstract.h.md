# src/xrGame/space_restriction_abstract.h

> The interface every restriction shares: it owns a *border* — the set of navigation vertices on its edge — and can name the subset of that border from which a step across is actually possible.

**Needs** — [`space_restriction_abstract_inline.h`](space_restriction_abstract_inline.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`space_restriction.cpp`](space_restriction.cpp.md) · [`space_restriction.h`](space_restriction.h.md) · [`space_restriction_abstract_inline.h`](space_restriction_abstract_inline.h.md) · [`space_restriction_base.h`](space_restriction_base.h.md) · [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md)
**Tier floor** — T2: walks the navigation mesh's neighbour lists; lazy, cached, and on the pathfinding path

## Purpose

An abstract base header, and therefore substantive: what it demands of an implementor is
the contract the whole restriction family satisfies. There are exactly two demands —
*produce a border*, and *answer your own name* — and one piece of shared machinery built
on top of them.

The border is the load-bearing idea of the entire subsystem. A restrictor is authored as a
geometric volume, but the pathfinder does not reason about volumes; it reasons about the
navigation mesh. So every restriction is reduced, once, to a **list of level-graph
vertices that lie on its edge**, and that list is then stamped as a mask onto the level
graph, which makes the search stop there. Everything else in the family — shapes,
compositions, bridges, the in/out pairing — exists to produce and combine border lists.

## State

```text
RECORD restriction_abstract
  border      : list<vertex>      # sorted; see space_restriction_base for the ordering
  initialized : bool              # the border has been built

  accessible_neighbour_border        : list<vertex>   # cached subset of `border`
  accessible_neighbour_border_actual : bool           # cache validity
```

**Invariants** — `border` is never empty once initialized; an empty border means the
restrictor's volume failed to touch the navigation mesh at all, which is an authoring
error and is asserted with the restrictor's name. The cached subset is computed at most
once and never invalidated — a border does not change after it is built, because the
volumes it comes from do not move.

## `initialize` (required of an implementor)

**Contract** — build this restriction's border, and set the initialized flag if it
succeeded. May legitimately *fail to complete*: a composition whose parts are not ready
yet returns without setting the flag, and the caller retries later. That two-outcome shape
is why the flag exists separately from the call.

## `name` (required of an implementor)

**Contract** — the restriction's identity: for a single restrictor its object name, for a
composition the normalized comma-joined list of its parts. Used as the cache key in the
holder and in every diagnostic.

## `border`

**Contract** — return the border, building it on first access. Hard-fails if the build did
not produce one. This lazy-on-read shape is why nothing in the level-load path has to
order restriction construction against navigation-mesh availability.

## `accessible_neighbour_border`

**Contract** — return the subset of the border from which a *legal step across the
boundary exists*: vertices that have at least one neighbour on the permitted side.
Computed on first call and cached. Hard-fails if the subset is empty, because a restriction
with a border but no way across is a trap that would strand any creature inside it.

```text
FUNCTION accessible_neighbour_border(restriction, is_out_restriction) -> list<vertex>
  IF NOT cached
    FOR EACH v IN border
      IF has_legal_neighbour(v)
        APPEND v TO cache
    cached = true
  REQUIRE cache IS NOT empty
  RETURN cache

FUNCTION has_legal_neighbour(v) -> bool
  FOR EACH n IN neighbours_of(v) IN the level graph
    IF n IS NOT a valid vertex   CONTINUE
    IF restriction.inside(n, partially = NOT is_out_restriction) IS is_out_restriction
      RETURN true
  RETURN false
```

**Notes** — the single condition in `has_legal_neighbour` covers both restriction senses
at once, and it repays reading slowly.

- For an **out** restriction (a volume the entity must stay inside), the legal side is
  *inside*, and the test demands the neighbour be *fully* inside — partial containment is
  not good enough, because stepping onto a vertex that only clips the volume would put the
  creature's body outside it.
- For an **in** restriction (a volume the entity must stay out of), the legal side is
  *outside*, and the test demands the neighbour be not even *partially* inside — again the
  stricter reading, for the same reason.

So the flag simultaneously selects which answer counts as legal and which strictness to
demand, and in both cases it picks the conservative one. A rebuild that splits this into
two separate predicates must keep both strictnesses.

The subset is used only by the nearest-accessible-position search, where restricting the
candidate set to vertices with a way across is what stops the search returning a point the
creature can reach but not leave.
