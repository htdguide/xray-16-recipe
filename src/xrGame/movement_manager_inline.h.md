# src/xrGame/movement_manager_inline.h

> The movement manager's accessors, and the three setters that quietly invalidate the path.

**Needs** — [`movement_manager.h`](movement_manager.h.md)
**Used by** — [`movement_manager.h`](movement_manager.h.md)
**Tier floor** — T3: field access, plus one invalidation rule

## Purpose

Mostly one-line accessors, separated out only because the original language wants inline
bodies after the class body; the split is arbitrary and a rebuild should fold them into
the type. Three of them are not accessors, and those carry the load.

## State

`Stateless` — it reads the manager's fields.

## Path invalidation on setting

**Contract** — `set_path_type` and `extrapolate_path(value)` each *conjoin* the actuality
flag with "the new value equals the old one" before storing the new value.

```text
FUNCTION set_path_type(new_type)
  actual := actual AND (current_type == new_type)   # unchanged value keeps the path
  current_type := new_type
```

**Invariants** — setting a field to the value it already holds must not invalidate the
path. This is the whole point of the pattern: brains re-issue the same movement order
every frame, and a naive setter would rebuild the path every frame with it. Any rebuild
that stores first and compares later has broken pathfinding performance across every
creature in the game.

## Subordinate accessors

**Contract** — `game_selector`, `game_path`, `level_path`, `detail`, `patrol`,
`restrictions`, `locations`, `object`, `level_path_builder` and `detail_path_builder` each
hand out one subordinate of the manager. Each asserts the subordinate was constructed; a
missing one is a construction-order bug, not a runtime condition, so the check exists only
in checked builds.

**Contract** — `base_game_params` and `base_level_params` hand out the two default search
cost evaluators, which the path managers adopt lazily on first use.

## Simple state readers

**Contract** — `actual`, `enabled`, `speed`, `old_desirable_speed`,
`wait_for_distributed_computation`, `extrapolate_path` and `body_orientation` read a field.
`set_desirable_speed` and `set_body_orientation` write one without touching actuality —
speed and facing do not change *which* path is correct.

**Contract** — `path_completed` is true only when the state machine has reached its
completed state *and* the path is still actual; a completed but stale path is not
completed.

**Contract** — `set_build_path_at_once` latches a flag that forbids handing any stage to a
worker thread. It is one-way: nothing clears it except reinitialization.

**Contract** — `accessible(position or vertex, radius)` forwards to the restrictor set.
