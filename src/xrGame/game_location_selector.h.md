# src/xrGame/game_location_selector.h

> Chooses where on the game graph a creature should wander next: either a search toward terrain matching its preferences, or a random walk that does not immediately double back.

**Needs** — [`abstract_location_selector.h`](abstract_location_selector.h.md) · [`location_manager.h`](location_manager.h.md) · [`game_location_selector_inline.h`](game_location_selector_inline.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — [`control_path_builder.cpp`](ai/monsters/control_path_builder.cpp.md) · [`game_location_selector_inline.h`](game_location_selector_inline.h.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md) · [`script_game_object.h`](script_game_object.h.md) · [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md)
**Tier floor** — T3: graph traversal and a weighted choice

## Purpose

The game-graph specialization of the generic location selector: given where a creature is
now, decide where it should head next across *levels*, not within one. Movement at this
scale is how the world stays populated — a creature that leaves the loaded level keeps
travelling — so this is the alife simulation's steering, not the pathfinder's.

The declaration lives here and the behaviour in
[`game_location_selector_inline.h`](game_location_selector_inline.h.md); the split is an
artifact of the original being a template, and a rebuild can merge them. Both are listed
because the mirror must be complete; read the behaviour in the inline twin.

## State

```text
RECORD GameLocationSelector       # extends the abstract selector over the game graph
  selection_type  : { mask, random_branching }
  previous_vertex : optional<graph_vertex>   # where we came from, to avoid doubling back
  locations       : LocationManager          # the creature's terrain preferences
```

**Invariants** — a location manager is required, not optional: the selector cannot decide
anything without the creature's terrain preference mask.

## `ESelectionType`

**Contract** — the two strategies.

- **Mask** — run the inherited graph search, whose evaluator scores vertices against the
  creature's preferred terrain. Used when the creature has somewhere it wants to be.
- **Random branching** — step to a randomly chosen neighbouring vertex that matches the
  preference mask, is on the current level and is reachable. Used when the creature is just
  moving.

The default after a reinitialisation is random branching, so a creature with nothing to do
wanders rather than standing still.

## the exported operations

- **construct** — takes the restricted object whose movement constraints apply, and the
  location manager supplying the terrain mask.
- **`reinit`** — resets the inherited state, selects random branching, and invalidates the
  previous-vertex memory (through the graph when one is supplied, so the invalid marker is
  the graph's own).
- **`set_selection_type`** / **`selection_type`** — read and write the strategy.
- **`actual`** — whether the current choice still stands.
- **`select_location`** — make a choice.
- **`select_random_location`** / **`accessible`** — the random walk and the reachability
  test; described in the inline twin.
