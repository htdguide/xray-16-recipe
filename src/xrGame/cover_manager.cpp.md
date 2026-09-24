# src/xrGame/cover_manager.cpp

> Builds the level's static cover point set from the navigation mesh at load time, and maintains the authored smart covers layered over it.

**Needs** — [`cover_manager.h`](cover_manager.h.md) · [`cover_manager_inline.h`](cover_manager_inline.h.md) · [`cover_point.h`](cover_point.h.md) · [`quadtree.h`](quadtree.h.md) · [`ai_space.h`](ai_space.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_storage.h`](smart_cover_storage.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrCore/Threading/ParallelFor.hpp`](../xrCore/Threading/ParallelFor.hpp.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)

**Used by** — reached through its declarations in [`cover_manager.h`](cover_manager.h.md); callers name that, not this file.
**Tier floor** — T2: a parallel sweep over a few hundred thousand navigation vertices at load time, feeding a spatial index; nothing here touches a device or a frozen layout

## Purpose

"Where can I stand so he cannot see me" is asked several times a second by every creature in
combat, and the level's navigation mesh already carries the answer in raw form: each
navigation vertex ships with a per-direction measure of how exposed it is, high and low.
This file turns that field of numbers into a sparse set of **cover points** — a few thousand
positions per level worth considering — and indexes them so the query is a radius search
rather than a sweep.

The reduction is the whole job. A level graph has hundreds of thousands of vertices; asking
an evaluator to score all of them within a radius would cost more than the decision is
worth. The selection rule implemented here keeps only the vertices where cover *begins*: the
positions at the edge of an obstruction, where stepping one cell further loses the
protection. Those are the corners a creature actually wants.

Smart covers are the second, authored layer: a designer places an object that declares named
firing positions with their own fields of view and animations. They live in the same index
as the computed points and are deliberately allowed to *delete* computed points near
themselves, so that a creature offered a hand-authored position is not distracted by a
mediocre generated one a metre away.

## State

```text
RECORD CoverManager
  covers                : quadtree<CoverPoint>   # the unified index: computed points and smart covers together
  temp                  : list<bool>             # per level-vertex scratch, valid only during the build
  nearest               : list<CoverPoint>       # reusable query result buffer, see Notes
  smart_covers_storage  : SmartCoverStorage      # the authored descriptions, keyed by table name
  smart_covers          : list<SmartCover>       # every instantiated smart cover
  smart_covers_actual   : bool                   # whether `smart_covers` is sorted by identifier
```

Invariants:

- **Every entry in the index is either a computed point or a smart cover**, distinguished by
  a flag on the point itself, and the two are released differently. Mixing them in one index
  is what lets a single radius query serve both kinds, and the flag is the price.
- **`smart_covers_actual` is false whenever a smart cover has been added** since the last
  sort. Lookup by identifier re-sorts lazily rather than keeping the list ordered on insert,
  because covers arrive in a burst at level load and are then queried for the level's life.
- **`temp` is meaningful only inside the build.** It is a per-vertex "this vertex has some
  cover and sits at an edge" bit, consumed immediately by the second pass.

**Notes** — the result buffer is a member rather than a local because the cover query runs
in the AI's inner loop and must not allocate. It is mutable so that the query can stay a
const operation, which is incidental; a rebuild wants the decision — *the query reuses one
scratch buffer and is therefore not safe to call from two threads at once* — and may solve
it with a per-caller buffer instead.

## `compute_static_cover`

**Contract** — called once when a level's navigation data is loaded. Discards any previous
cover set, sizes a spatial index to the level's bounding box, classifies every navigation
vertex in parallel, then inserts the surviving ones as cover points. Finally creates the
(empty) smart-cover storage. Allocates heavily; blocks for the duration; runs before any
creature exists.

**Invariants** — afterwards the index holds only computed points, and the smart-cover
storage exists and is empty. Calling it twice is safe because it clears first.

```text
FUNCTION compute_static_cover()
  clear()
  covers = new quadtree(
      bounds    = level_graph.bounding_box,
      leaf_size = level_graph.cell_size / 2,     # see Notes
      capacity  = 8 * 65536,
      nodes     = 4 * 65536)
  temp.resize(level_graph.vertex_count)

  # Pass 1 — parallel, read-only over the graph, one write per vertex
  FOR EACH i IN 0 .. level_graph.vertex_count   # split into ranges across workers
    v = level_graph.vertex(i)
    IF sum of v.high_cover over the four directions > 0
      temp[i] = is_edge_vertex(i)
    ELSE IF sum of v.low_cover over the four directions > 0
      temp[i] = is_edge_vertex(i)
    ELSE
      temp[i] = false

  # Pass 2 — serial, because it reads neighbours' pass-1 results and inserts into the index
  FOR EACH i IN 0 .. level_graph.vertex_count
    IF temp[i] AND is_critical_cover(i)
      covers.insert(CoverPoint(level_graph.position_of(i), i))

  smart_covers_storage = new SmartCoverStorage()
```

**Notes** — the two passes cannot be merged. Pass 2's criticality test reads `temp` for
vertices *two links away* from the one being tested, so it needs the whole of pass 1
finished. Pass 1 is the expensive half (it touches every vertex and its four links) and is
the one that is parallelised; pass 2 is serial both because of that dependency and because
the index does not accept concurrent insertion.

The quadtree is sized with leaf cells at half the navigation cell size — finer than the
data, so that a leaf holds at most one cover point in the dense case — and with fixed
capacities of 524288 points and 262144 nodes. Those are hard ceilings, not hints: a level
that produced more cover points than that would overflow. They are generous for the shipped
levels by a wide margin, and a rebuild is better off growing the index than reproducing the
numbers.

The high/low distinction is the creature's stance: high cover protects someone standing, low
cover only someone crouched. The build accepts a vertex on the strength of *either*, and
loses the distinction at that point — which stance a given point supports is re-derived by
the evaluator later from the same graph data.

## `edge_vertex`

**Contract** — private. True when the vertex has, in at least one of the four cardinal
directions, **no neighbouring navigation vertex** and **a cover value below the threshold**
in that same direction, for either stance. Pure; reads the level graph.

**Invariants** — "no neighbour" here means the graph link is invalid, which on a navigation
mesh means the walkable surface ends — a wall, a drop, or the mesh boundary.

**Notes** — the test is the heart of the reduction and it reads backwards at first. A vertex
is interesting when the direction that is *blocked for walking* is also the direction with
*low cover*. That is the definition of standing at the open side of an obstruction: you
cannot walk that way because something is there, and the cover field says you are exposed
that way because you are past its edge. A vertex buried inside cover fails the test (it has
high cover in the blocked direction); a vertex in the open fails it too (it has neighbours
in every direction).

The threshold is 16 on the shipped cover scale. Below that counts as "not covered". The
number is a tuning constant with no derivation in the source; it is the one value in this
file a rebuild cannot justify from first principles, only reproduce.

## `critical_cover` / `critical_point` / `cover`

**Contract** — private, and together they form the second filter: of the edge vertices, keep
only those that sit at a **corner** of the obstruction rather than along its flat face.
Pure; read the level graph and `temp`.

```text
FUNCTION is_critical_cover(index)
  v = level_graph.vertex(index)
  # test each cardinal direction against the two perpendicular to it
  RETURN is_critical_point(v, 0, 1, 3)
      OR is_critical_point(v, 2, 1, 3)
      OR is_critical_point(v, 1, 0, 2)
      OR is_critical_point(v, 3, 0, 2)

FUNCTION is_critical_point(v, blocked_dir, side_a, side_b)
  IF v has a neighbour in blocked_dir: RETURN false     # nothing blocks here
  RETURN v has no neighbour in side_a                   # the obstruction turns a corner
      OR v has no neighbour in side_b
      OR diagonal_is_edge(v, side_a, blocked_dir)       # the neighbour around the corner
      OR diagonal_is_edge(v, side_b, blocked_dir)       #   is itself an edge vertex

FUNCTION diagonal_is_edge(v, first, second)
  n = v.neighbour(first)
  IF n is none: RETURN false
  d = n.neighbour(second)
  IF d is none: RETURN false
  RETURN temp[d]                                        # pass-1 result for the diagonal
```

**Notes** — the shape being detected is "the obstruction ends here, or bends here". A long
straight wall produces edge vertices along its entire length, and keeping them all would
flood the index with interchangeable positions; keeping only the ends and the corners keeps
the positions a creature can actually *use* — the ones it can lean out from. The diagonal
test is what catches an obstruction that ends one cell over, which a purely local test at
this vertex would miss.

This is the pass that depends on `temp` for other vertices, and hence the reason for the
two-pass structure.

## `add_smart_cover`

**Contract** — instantiates one authored smart cover from a named description in the
storage, given the placed object, whether it counts as combat cover, whether it may be fired
from, and the loophole table supplied by the script layer. Removes competing computed points
around it, inserts it into the shared index, records it and invalidates the sort. Returns
the new cover. Allocates. Called during level setup, from the script side.

```text
FUNCTION add_smart_cover(table_name, object, is_combat_cover, can_fire, loopholes)
  cover = new SmartCover(object, storage.description(table_name), is_combat_cover, can_fire, loopholes)
  remove_nearby_covers(cover, object)
  covers.insert(cover)
  smart_covers.append(cover)
  smart_covers_actual = false
  RETURN cover
```

## `remove_nearby_covers`

**Contract** — private. For each loophole of a newly placed smart cover, finds the computed
points within the placed object's radius plus one unit of that loophole's field-of-view
origin, discards any that lie **outside** the object's volume, and permanently removes and
releases the rest from the index. Never removes another smart cover.

```text
FUNCTION remove_nearby_covers(cover, object)
  FOR EACH loophole IN cover.loopholes
    origin = cover.fov_position(loophole)
    found  = covers.nearest(origin, object.radius + 1)
    keep only entries that are NOT smart covers AND whose position is inside object
    FOR EACH p IN found: covers.remove(p)
    release found
```

**Notes** — the "plus one unit" and the inside-the-object test pull in opposite directions
and that is intentional: search a little wider than the object so a point sitting exactly on
its boundary is considered, then delete only what is genuinely *within* the authored volume.
The result is that a designer placing a smart cover clears the generated points it stands
on, and nothing else — a generated point just outside the barricade survives, because it is
a real alternative.

The deletion is permanent for the level's lifetime. Nothing restores computed points if a
smart cover is removed, and nothing needs to: smart covers are placed at level setup and
persist.

## `smart_cover` (lookup by identifier)

**Contract** — finds an instantiated smart cover by its placed object's name. Sorts the list
on first use after any insertion, then binary-searches. Fails loudly when the identifier is
unknown, because the caller is script naming an authored object — a miss is a data error,
not a runtime condition.

**Notes** — the ordering key is the interned string's *identity*, not its text. That is
sound only because the string store guarantees one instance per distinct text, and it makes
both the comparison and the equality check pointer-cheap. A rebuild without interning must
compare text and will pay for it, or key the lookup by a hash instead.

## `clear`

**Contract** — empties the index, releasing every entry (computed points and smart covers
release differently, so the type flag is consulted), drops the smart-cover storage and
forgets the smart-cover list. Safe when nothing has been built.

**Notes** — the index itself is kept and only emptied, while the storage is destroyed and
recreated by the next build. The asymmetry is incidental. What is load-bearing is that
every cover point is owned by exactly one place — this index — and that a level transition
must release all of them, because both kinds hold references into level-specific data.
