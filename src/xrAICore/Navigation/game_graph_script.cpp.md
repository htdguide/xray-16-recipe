# src/xrAICore/Navigation/game_graph_script.cpp

> Exposes the cross-level graph to the script layer — the part of the navigation surface mods are allowed to see.

**Needs** — [`game_graph.h`](game_graph.h.md) · [`../AISpaceBase.hpp`](../AISpaceBase.hpp.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a registration table. It is only in the manual tier because the objects it hands out are views into a mapped file.

## Purpose

Shipped game scripts query the cross-level graph directly — to ask which level a vertex is on, to
measure a coarse distance between two places, to block a vertex. This file is the entire list of
what they may do. Because every shipped script must keep running unmodified (conformance
criterion 10), the *names and signatures here are frozen*: a rebuild may implement them however
it likes but must export exactly these names with exactly these argument shapes.

## Stateless.

## The exported surface

```text
game_graph()                        -> the one graph, reached through the global AI space
graph.accessible(vertex_id)         -> bool          # is the vertex open
graph.accessible(vertex_id, value)                   # open or close it
graph.valid_vertex_id(vertex_id)    -> bool
graph.vertex(vertex_id)             -> GameVertex
graph.vertex_id(vertex)             -> int
graph.levels()                      -> iterable of (level_id, Level)

vertex.level_point()                -> vector3       # position within its own level
vertex.game_point()                 -> vector3       # position in world space
vertex.level_id()                   -> int
vertex.level_vertex_id()            -> int

gg_vertex_level_id(vertex_id)       -> int           # the level a game vertex belongs to
gg_level_id(index)                  -> int           # the level id at an index in the table
gg_levels_count()                   -> int
gg_distance(vertex_id_a, vertex_id_b) -> real        # straight-line, world space
```

**Contract** — `game_graph()` and the four `gg_`-prefixed functions reach the graph through the
global AI-space handle rather than taking it as an argument, because script code has no way to
hold one. The two vertex position accessors refuse a missing vertex with a script-visible error
rather than faulting.

**Notes** — three things in this list are worth a rebuilder's attention.

`gg_distance` is *not* the graph's edge cost. It is the straight-line distance between two
vertices' world positions, computed on the spot, and it works for any two vertices whether or not
they are adjacent. The graph's own `distance` is the stored path cost of a single edge and fails
on non-adjacent vertices. Scripts use the first for "roughly how far away is that place", and
confusing the two in a rebuild silently changes AI behaviour everywhere.

`gg_level_id` takes a *position in the level table*, not a level identifier, and returns the
identifier stored there. The two coincide in shipped data only by accident of how levels are
numbered, and script code relies on the indexed form.

The level table is exported as an iterable of pairs so that a script can walk every level in the
game; the pair type is exported under its own name purely so the iteration has something to yield.
