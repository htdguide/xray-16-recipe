# src/xrGame/smart_cover_description.h

> Declares the shared, authored shape of one *kind* of smart cover: its loopholes and the graph of transitions between them.

**Needs** — [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`xrAICore/Navigation/graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md)
**Used by** — [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_description.cpp`](smart_cover_description.cpp.md) · [`smart_cover_description_inline.h`](smart_cover_description_inline.h.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`smart_cover_object.cpp`](smart_cover_object.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`smart_cover_storage.cpp`](smart_cover_storage.cpp.md) · [`stalker_movement_params.cpp`](stalker_movement_params.cpp.md)
**Tier floor** — T2: a loaded graph shared between instances

## Purpose

Declares the surface implemented in
[`smart_cover_description.cpp`](smart_cover_description.cpp.md) and
[`smart_cover_description_inline.h`](smart_cover_description_inline.h.md).

A description is the *template*: "a waist-high wall with three firing positions and these
ways of moving between them". Many placed covers on a level share one description, which
is why it is reference-counted and age-stamped
([`smart_cover_storage.h`](smart_cover_storage.h.md) evicts the unused ones) rather than
owned by a cover.

## Exported units

- **The class** — loopholes plus a transition graph, both built from one authored table.
- **`table_id`** — the authored name this was loaded from; the identity used in every
  failure message.
- **`loopholes`** — all loopholes of this kind, in authored order.
- **`transitions`** — the graph whose vertices are loophole names (plus the two reserved
  enter/exit names) and whose edges carry a weight and a list of transition actions.
- **`get_loophole`** — find a loophole by name; yields nothing when absent.

## State

```text
RECORD description
  loopholes   : list<loophole>
  transitions : graph
      vertex_id   : text                 # loophole name, or a reserved enter/exit name
      edge_weight : real                 # search cost
      edge_data   : list<transition_action>
  table_id    : text
```

**Invariants** — enforced at load: at least one loophole; at least one *usable*; at least
one reachable from the enter vertex; at least one with an edge to the exit vertex. A
description violating any of these describes a cover a creature could enter and never
leave, or never enter at all.
