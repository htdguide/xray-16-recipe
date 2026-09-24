# src/xrGame/movement_manager_level.cpp

> Drives the path pipeline when the destination is a vertex of the loaded level: search the navigation mesh, smooth it, follow it.

**Needs** — [`movement_manager.h`](movement_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`detail_path_builder.h`](detail_path_builder.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`mt_config.h`](mt_config.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: graph search orchestration

## Purpose

The common case: the destination is already a level vertex, so the coarse graph is not
involved and the pipeline is two searches deep instead of three. Separated from its game
and patrol siblings by path kind only.

## State

`Stateless.`

## `process_level_path`

**Contract** — advances the path state machine by one step for the level-path kind. Same
non-blocking discipline as the game-path driver: a stage that may be deferred is queued
with the worker pool and the state is left for the next update to resume.

```text
FUNCTION process_level_path()
  IF level path stale AND state past build_level_path THEN state := build_level_path

  STEP build_level_path
    configure the level path builder with (our level vertex, the destination vertex,
       whether to extrapolate, no explicit target position)
    IF this stage may be deferred THEN queue the builder; BREAK
    run the level search inline
    IF the caller did not demand an at-once build THEN BREAK
    fall through                            # at-once builds run the whole pipeline now

  STEP continue_level_path
    pick the next intermediate vertex
    fall through

  STEP build_detail_path
    detail path follows patrol-style iff extrapolation is on
    start position := our position; start heading := negated body yaw
    configure the detail builder with the level path and the intermediate index
    IF this stage may be deferred THEN queue it; BREAK
    run the detail build inline; BREAK

  STEP verification
    level path stale   -> build_level_path
    detail path stale  -> build_level_path
    detail path consumed -> continue_level_path,
       and if the level path is consumed too -> completed

  STEP completed
    either path stale -> build_level_path
```

**Invariants** — unlike the game-path driver, this one *falls through* from the level
search into the detail build when an at-once build was demanded. That is the whole meaning
of the at-once flag: a caller that cannot tolerate a partial path across update boundaries
— a scripted forced move, a spawn placement — gets a complete, walkable detail path before
the call returns.

**Notes** — no destination position is handed to the level builder here, where the game and
patrol drivers both supply one. The level pipeline's destination *is* a vertex, so the
vertex's own position is the target; the other two have a target that sits between or past
vertices and must be named separately.

Whether the detail path is patrol-style is tied to the extrapolation flag rather than
chosen independently, because both answer the same underlying question: may the creature
walk past the last level vertex, or must it stop exactly on it.
