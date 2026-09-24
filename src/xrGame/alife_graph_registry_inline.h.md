# src/xrGame/alife_graph_registry_inline.h

> Moving an object between graph vertices, seeding a creature's travel along a graph edge, and the registry's accessors.

**Needs** — [`alife_graph_registry.h`](alife_graph_registry.h.md)
**Used by** — [`alife_graph_registry.cpp`](alife_graph_registry.cpp.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md)
**Tier floor** — T3: index bookkeeping and one edge lookup.

## Purpose

Two of the operations here carry real decisions — how a move between graph vertices is
expressed, and what it means for a creature to be *between* two vertices — and the rest are
accessors.

## `change`

**Contract** — Moves an object from one graph vertex to another. Requires the object to be
one that uses navigation locations. Remove, add, then rewrite the object's own placement
fields from the destination vertex.

```text
FUNCTION change(object, from_vertex, to_vertex)
  REQUIRE object uses navigation locations
  remove(object, from_vertex)
  add(object, to_vertex)
  object.graph_vertex = to_vertex
  object.position     = to_vertex's own level point
  object.level_vertex = to_vertex's own level vertex
```

**Invariants** — A graph move *is* a teleport to the destination vertex's representative
point. That is the reason every caller that has computed a better fine position saves and
restores it around this call (see [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md)
and [`alife_anomalous_zone.cpp`](alife_anomalous_zone.cpp.md)). A rebuild should offer a
second form that moves the index without rewriting the placement, and delete the dance from
three call sites.

## `assign`

**Contract** — Initializes a creature's graph-travel state so it is treated as standing *at*
its current vertex rather than in transit. Sets both the previous and next vertex to the
current one, and derives how far it still is from the next.

```text
FUNCTION assign(creature)
  creature.next_vertex = creature.previous_vertex = creature.graph_vertex
  creature.distance_to_point = creature.distance
  FOR EACH outgoing edge of the creature's vertex, in the graph's own order
    IF the edge's length exceeds the creature's distance
      creature.distance_from_point = edge length - creature.distance
      BREAK
```

**Invariants** — A creature's offline position is a vertex plus a distance along an edge, so
two distances are needed: how far it is past the previous vertex and how far it still is
from the next. The first is copied; the second is derived from the *first edge long enough
to contain it*. That is an approximation — the edge chosen is whichever the graph lists
first, not the one the creature is actually travelling — and it is acceptable only because
the value seeds a coarse simulation that re-derives it as soon as the creature moves. If no
edge is long enough, the field is left at whatever it held.

## Accessors

**Contract** — `level` returns the level subset and asserts it exists; `actor` returns the
player's record as optional, since it is absent before the player is registered; `objects`
returns the whole per-vertex index; `set_process_time` records the simulation's time budget
and forwards it to the level subset when one exists.

**Contract** — `iterate_objects` visits every object at one graph vertex by handing a
callback to the mutation-safe table, which is the only permitted way to walk one: the
callbacks routinely add and remove objects at the vertex they are walking.
