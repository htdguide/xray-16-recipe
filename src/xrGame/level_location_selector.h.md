# src/xrGame/level_location_selector.h

> Declares the level-graph flavour of "find me the best nearby place to stand", by specializing the generic location selector for the fine navigation mesh.

**Needs** — [`abstract_location_selector.h`](abstract_location_selector.h.md) · [`level_location_selector_inline.h`](level_location_selector_inline.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`control_path_builder.cpp`](ai/monsters/control_path_builder.cpp.md) · [`control_path_builder_base.cpp`](ai/monsters/control_path_builder_base.cpp.md) · [`level_location_selector_inline.h`](level_location_selector_inline.h.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

The generic location selector searches outward from a starting vertex on *some* graph,
scoring every vertex it reaches and keeping the best. This file names the level graph as
that graph, and overrides the two hooks the generic search offers — one before the search
and one after. See
[`level_location_selector_inline.h`](level_location_selector_inline.h.md) for what those
hooks do, which is the whole of the file's content.

Exported units:

- the level-graph specialization of the location selector, parameterized by the scoring
  function and the vertex identifier width.
- `before_search` — snap the start inside the restrictors and widen them.
- `after_search` — restore the restrictors.

**Notes** — the generic selector and its two graph specializations (this one and the game
graph's) are a compile-time family in the original. What a rebuild needs is one search with
a pluggable graph, a pluggable score, and a pair of enter/leave hooks; nothing about the
family needs to be resolved at compile time except performance.
