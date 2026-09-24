# src/xrAICore/Navigation/PathManagers/path_manager_params_game_vertex.h

> The request that confines a cross-level search to vertices whose terrain matches a creature's preferences.

**Needs** — [`path_manager_params.h`](path_manager_params.h.md) · [`../game_graph_space.h`](../game_graph_space.h.md)
**Used by** — [`path_manager_game_vertex.h`](path_manager_game_vertex.h.md) · [`path_manager_game_vertex_inline.h`](path_manager_game_vertex_inline.h.md)
**Tier floor** — T2: a request record holding a borrowed preference table.

## Purpose

Every vertex of the cross-level graph carries a small vector of terrain attributes — what kind
of place it is. Each creature kind carries a table of masks describing which places it will
cross. This request pairs a search with such a table so that one creature's route avoids
water and another's avoids open ground, without either needing its own graph.

## State

```text
RECORD GameVertexRequest EXTENDS SearchLimits
  vertex_types           : ref to list<TerrainMask>  # BORROWED; the creature's preference
                                                     #   table, not copied
  vertex_id              : int                 # OUT: reset to the invalid marker at setup
  path                   : optional<list<int>> # OUT: the route
  max_range              : real  # default 6000
  max_iteration_count    : int   # default: no limit
  max_visited_node_count : int   # default: no limit
```

**Invariants** — the preference table is referenced, not owned. It must outlive the search, and
it must not be edited while the search runs. An empty table means no vertex matches, which the
policy treats as a mistake worth a warning rather than a legitimate request.

**Notes** — the vertex-identifier output field is present and reset but never written by the
policy that uses this request; only the accessibility filter is. It is inherited from the
sibling request shape above. A rebuild should drop it.
