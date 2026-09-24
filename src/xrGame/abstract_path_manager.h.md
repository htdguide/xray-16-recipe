# src/xrGame/abstract_path_manager.h

> Declares the reusable "find and hold a path to a known destination" component whose behaviour is written in [`abstract_path_manager_inline.h`](abstract_path_manager_inline.h.md).

**Needs** — [`restricted_object.h`](restricted_object.h.md) · [`abstract_path_manager_inline.h`](abstract_path_manager_inline.h.md)
**Used by** — [`abstract_path_manager_inline.h`](abstract_path_manager_inline.h.md) · [`game_path_manager.h`](game_path_manager.h.md) · [`game_path_manager_inline.h`](game_path_manager_inline.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`level_path_manager_inline.h`](level_path_manager_inline.h.md)
**Tier floor** — T3: a declaration; the parameterization over graph, scorer and index is a language convenience

## Purpose

Declares the path manager: the half of navigation that is given a destination and must
produce and maintain a route to it. It is written once against an unspecified graph,
vertex identity, edge scorer and index type, so that the same component serves the fine
level graph, the coarse game graph and the patrol-path graph. Its twin component, which
*chooses* a destination instead of being given one, is
[`abstract_location_selector.h`](abstract_location_selector.h.md).

Substance — the actuality rules, the negative cache and the intermediate-vertex
convention — is in
[`abstract_path_manager_inline.h`](abstract_path_manager_inline.h.md).

Exported units:

- `reinit` — reset and bind a graph.
- `set_evaluator`, `evaluator` — bind and read the edge scorer; rebinding invalidates.
- `set_dest_vertex`, `dest_vertex_id` — the destination; changing it invalidates.
- `actual`, `make_inactual` — the "does my path still stand" flag.
- `path`, `intermediate_index`, `completed` — the produced route and how far along it the
  owner has committed.
- `select_intermediate_vertex` — choose how much of the path to commit to next.
- `failed`, `reset`, `invalidate_failed_info` — failure state and the negative cache.
- `object` — the creature whose movement restrictions bound every search.
- `build_path`, `before_search`, `after_search`, `check_vertex` — hooks a concrete manager
  overrides.

**Notes** — the movement manager is granted direct access to this component's internals
rather than going through the accessors. That is a coupling, not an interface: the
movement manager is the only legitimate driver of a path manager, and in a rebuild the two
should be one module or the access should be an explicit privileged view.
