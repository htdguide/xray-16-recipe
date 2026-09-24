# src/xrGame/space_restriction_bridge_inline.h

> The nearest-legal-position search: from an illegal point, find the closest navigation vertex on the legal side of a restriction and a point inside that cell to actually stand on.

**Needs** — [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_base.h`](space_restriction_base.h.md) · [`space_restriction_abstract.h`](space_restriction_abstract.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`space_restriction_bridge.cpp`](space_restriction_bridge.cpp.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md)
**Tier floor** — T2: a linear scan over a cached vertex subset plus a neighbour scan; called when a goal turns out to be unreachable, not every frame

## Purpose

Scripts and the alife simulation hand the movement system destinations that are not
necessarily legal: a smart terrain's job location outside a creature's permitted region, a
patrol point inside a zone it may not enter. Rather than fail, the movement system asks for
the nearest position it *may* occupy, and this is that search.

It is written generically over the restriction whose border and containment test to consult,
because the caller is sometimes the bridge's own implementation and sometimes an entire
composed restriction that subtracts one volume from another. The same three-stage search
serves both.

## `accessible_nearest`

**Contract** — takes a query position and a flag saying which sense of restriction is being
satisfied; returns the nearest legal navigation vertex and writes out a legal point within
it. Returns the invalid vertex identifier if no legal vertex is found. Reads the level
graph and the restriction's cached crossable-border subset; allocates nothing. Hard-fails
in checked builds on an uninitialized restriction or an empty border.

```text
FUNCTION accessible_nearest(restriction, position, is_out_restriction) -> (vertex, point)
  REQUIRE restriction.initialized
  candidates = restriction.accessible_neighbour_border(is_out_restriction)
  REQUIRE candidates IS NOT empty

  # 1. Nearest border vertex. The border is the cheap index into "near the boundary";
  #    it is a small list, and scanning it beats searching the mesh.
  seed = argmin over candidates OF squared_distance(vertex_position(c), position)
  IF seed is not a valid vertex  RETURN invalid

  # 2. Step one cell to the legal side. A border vertex is ON the boundary and is
  #    itself not somewhere a body may stand; its neighbours include cells that are.
  best = none
  FOR EACH n IN neighbours_of(seed) IN the level graph
    IF n is not a valid vertex  CONTINUE
    IF restriction.inside(n, partially = NOT is_out_restriction) IS NOT is_out_restriction
      CONTINUE
    keep n if it is closer to position than the current best
  IF best is none  RETURN invalid

  # 3. A point inside that cell, not merely its centre: pick the corner or centre
  #    closest to where the caller wanted to be.
  point = nearest_sample_in_cell(best, position)
  RETURN (best, point)

FUNCTION nearest_sample_in_cell(v, position) -> point
  centre = vertex_position(v)
  half   = navigation cell size / 2 - tiny
  samples = the four corners at (centre.x ± half, centre.z ± half), each lifted to
            the cell's own sloped plane, plus the centre itself
  RETURN the sample nearest to position
```

**Invariants**

- The legality test in stage 2 is the same composed condition the crossable-border cache
  uses, and it carries both meanings at once. For an **out** restriction the legal side is
  inside and the neighbour must be *fully* inside; for an **in** restriction the legal side
  is outside and the neighbour must not be even *partially* inside. Both are the
  conservative reading, so the returned cell is one a body fits in rather than one it
  merely overlaps.
- The candidate set is the *crossable* subset of the border, not the whole border. A border
  vertex with no legal neighbour is a dead end — stage 2 would fail there and the whole
  search would return nothing even though a legal position exists two cells away. Filtering
  the candidates up front, once, is what makes stage 2's single-step scan sufficient.
- Corners are pulled in by an epsilon so the returned point belongs unambiguously to the
  cell it was derived from and not to its neighbour.

**Notes** — the search is deliberately local: one nearest-border-vertex scan and one
neighbour step, not a mesh search. It answers "just outside the wall, near where you
wanted" rather than "the globally closest legal cell", and the two differ when the boundary
is concave. That is accepted, because the caller immediately runs a real path search from
the result and a slightly wrong starting cell costs a slightly longer path.

Choosing among five points in the final cell rather than returning its centre matters for
the same reason: the caller usually wants to stand as close to an object beyond the border
as it can, and half a cell is about half a body width.

The original marks this routine as a candidate for optimization if it ever shows in a
profile; the obvious win is that the crossable-border subset is fetched three times per
call, each fetch re-checking the cache flag.

## `accessible_neighbour_border`

**Contract** — forwards to the implementation's cached crossable-border subset. See
[`space_restriction_abstract.h`](space_restriction_abstract.h.md) for how that subset is
computed.

## Constructor and `object`

**Contract** — the cell is constructed around an implementation, which may not be absent,
and `object` hands it back. Every caller of `object` is inside the family; nothing outside
holds the result.
