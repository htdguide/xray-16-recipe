# src/xrGame/game_path_manager_inline.h

> Walking a game-graph path: the index starts at the second vertex, not the first, because the creature is already standing on the first.

**Needs** — [`game_path_manager.h`](game_path_manager.h.md) · [`abstract_path_manager.h`](abstract_path_manager.h.md)
**Used by** — [`game_path_manager.h`](game_path_manager.h.md)
**Tier floor** — T3: index arithmetic over a path

## Purpose

Three small decisions that together define how a cross-level path is consumed. Everything
else is delegation to the generic path manager.

## `actual`

**Contract** — the held path is still valid when the generic test passes for the creature's
*current* game-graph vertex against the path's destination. The creature's position is read
from its AI location record, not from the path's own start, so a path becomes stale the
moment the creature is somewhere the path does not begin.

## `select_intermediate_vertex`

**Contract** — advance the index into the path.

```text
FUNCTION advance()
  FAIL WITH empty_path IF the path is empty
  IF the index is already set THEN index = index + 1
  ELSE IF the path has fewer than two vertices THEN index = 0
  ELSE index = 1
```

**Invariants** — the first advance lands on the **second** vertex. The first vertex of a path
is where the creature already is, so aiming at it would be a no-op step; the only exception
is a single-vertex path, which has no second vertex and must aim at its only one. This is
the decision the whole file exists for.

## `completed`

**Contract** — the path is complete when the generic test says so *and* the index has reached
the last vertex. A path still short of its last vertex is never complete, whatever the
generic test says.

**Invariants** — the extra condition guards against the generic test's notion of arrival —
proximity to the destination — declaring success while intermediate vertices remain. On the
game graph, the intermediate vertices are level transitions that must actually be traversed;
being near the destination in space is not the same as having got there.

## `before_search` / `after_search`

**Contract** — both empty. A game-graph search needs no bracket.
