# src/xrGame/level_location_selector_inline.h

> Makes a best-place-to-stand search on the fine navigation mesh respect the searcher's restrictors — and temporarily widen them so the answer is reachable.

**Needs** — [`level_location_selector.h`](level_location_selector.h.md) · [`abstract_location_selector.h`](abstract_location_selector.h.md) · [`restricted_object.h`](restricted_object.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`level_location_selector.h`](level_location_selector.h.md)
**Tier floor** — T3: a graph search's entry and exit hooks

## Purpose

A creature asking "where is the best place near here to stand" — for cover, for a firing
position, for an ambush — must not be handed an answer it is forbidden to walk to. This
file is the two-line answer: before the search, make sure the search *starts* somewhere
legal, and widen the legal region by the search's own radius; after it, restore.

It is separate from the generic selector because only the level graph has restrictors; the
game graph, which the alife simulation searches, has none.

## State

`Stateless.` — everything is reached through the search's own fields: the graph, the scoring
function, the result path, and the restricted object doing the searching.

## `before_search`

**Contract** — called once with the intended start vertex, which it may **replace**. Does
nothing at all when the searcher has no restrictors. Otherwise: if the start vertex is
forbidden, the start is moved to the nearest permitted vertex; then the restrictor set is
temporarily widened by a border of the scoring function's own radius.

```text
FUNCTION before_search(INOUT start_vertex)
  IF the searcher has no restrictors THEN RETURN
  IF start_vertex is not accessible THEN
    start_vertex = nearest accessible vertex to its position
  add a border around start_vertex of the evaluator's radius
```

**Invariants** — the border must be removed by the matching `after_search`, and the two
always pair. A search that returns early between them leaves the creature's restrictors
permanently widened, which is invisible until it walks somewhere it should not.

**Notes** — the two decisions are independent and both are load-bearing.

*Snapping the start*: a creature standing at a vertex it is not allowed to occupy is a real
and common state — it was pushed, it was teleported, its restrictors just changed. Searching
from a forbidden vertex would find nothing, because the first step is already refused, and
the creature would freeze. Snapping means the search always has somewhere to begin.

*Widening by the radius*: the scoring function has a radius — the area it considers around
each candidate — and a candidate at the very edge of the permitted region has part of that
area outside it. Without the widening the search cannot even *look* at those vertices, and
the creature never uses the edge of its own allowed space, which is exactly where cover
usually is. The border is a temporary relaxation for looking, not for walking: the resulting
path is still built under the unwidened restrictors.

## `after_search`

**Contract** — removes the border added by `before_search`, restoring the searcher's true
restrictors. Does nothing when the searcher has none. Must run on every exit path from the
search, successful or not.
