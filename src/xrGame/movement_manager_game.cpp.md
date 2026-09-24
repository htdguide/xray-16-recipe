# src/xrGame/movement_manager_game.cpp

> Drives the path pipeline when the destination is on another level: pick a game vertex, search the coarse graph, then resolve one level-sized leg at a time.

**Needs** — [`movement_manager.h`](movement_manager.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_level_registry.h`](alife_level_registry.h.md) · [`game_location_selector.h`](game_location_selector.h.md) · [`game_path_manager.h`](game_path_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`detail_path_builder.h`](detail_path_builder.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`mt_config.h`](mt_config.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: graph search orchestration, no device or format contact

## Purpose

The longest of the three pipeline drivers, because a game path is the only one that can
cross a level boundary. It turns an abstract destination ("somewhere that satisfies this
evaluator, anywhere in the world") into a concrete detail path inside the currently loaded
level, and recognizes the moment the creature must leave the level entirely.

It is a separate file from its level and patrol siblings purely by path kind; all three
are branches of one state machine whose shared parts live in
[`movement_manager.cpp`](movement_manager.cpp.md).

## State

`Stateless` — every field it touches belongs to the movement manager.

## `process_game_path`

**Contract** — advances the path state machine by one step, for the game-path kind. Called
once per creature update from `update_path`. Never blocks: when a search is too expensive
it registers a builder with the worker pool and returns, leaving the state unchanged so
the next call resumes. It may spawn a cross-level transition, which destroys the creature's
presence on this level.

**Invariants** — the pipeline is strictly ordered and the order is load-bearing:

```text
select game vertex -> build game path -> continue game path
  -> build level path -> continue level path -> build detail path
  -> verification -> completed
```

Before any step runs, staleness is propagated *downward only*: if the level path went
stale and the machine is past the build-level-path step, it falls back to that step; if the
game path went stale and the machine is past build-game-path, it falls back further. A
stale coarse result always invalidates the fine results derived from it, never the reverse.
The teleport state is exempt from both checks — once a cross-level move has been ordered,
nothing may rewind it.

```text
FUNCTION process_game_path()
  IF state is not teleport
    IF level path stale AND state past build_level_path THEN state := build_level_path
    IF game path stale  AND state past build_game_path  THEN state := build_game_path

  # each step falls through into the next on success; a failure or a deferral breaks out
  STEP select_game_vertex
    ask the game location selector for a destination vertex, seeded from where we are
    IF the answer is unchanged AND the selector has already been consulted
      state := completed; BREAK            # we are already standing at the best place
    IF the selector failed THEN BREAK      # try again next update
    fall through

  STEP build_game_path
    search the coarse graph from our game vertex to the destination
    IF failed THEN report the diagnosis and BREAK
    fall through

  STEP continue_game_path
    pick the next intermediate vertex along the coarse path
    IF that vertex lies on a different level
      state := teleport; hand the creature to the cross-level transition; BREAK
    fall through

  STEP build_level_path
    target := the level vertex under the intermediate game vertex
    IF that vertex is outside our restrictors
      target := nearest restrictor-legal vertex to its position
    configure the level path builder with (our level vertex, target, extrapolate, target position)
    IF this stage may be deferred THEN queue the builder; BREAK
    run the level search inline; BREAK

  STEP continue_level_path
    pick the next intermediate vertex along the level path
    fall through

  STEP build_detail_path
    tell the detail path manager it is following a patrol-style path,
      its start position and start heading (taken from the body's current yaw, negated),
      and its destination (the intermediate level vertex's position)
    configure the detail path builder with the level path and the intermediate index
    IF this stage may be deferred THEN queue the builder; BREAK
    run the detail build inline; BREAK

  STEP verification                        # the steady state while walking
    re-ask, cheapest question first:
      selector no longer valid for our position -> select_game_vertex
      game path stale                           -> build_game_path
      level path stale                          -> build_level_path
      detail path stale                         -> build_level_path   # not build_detail_path
      detail path consumed                      -> continue_level_path,
        and if the level path is also consumed  -> continue_game_path,
        and if the game path is also consumed   -> completed

  STEP completed
    only one question remains: is the chosen destination still the best one?
    if not -> select_game_vertex

  STEP teleport
    do nothing; the transition owns the creature now
```

**Notes** — a stale *detail* path rewinds to build-**level**-path, not to build-detail-path.
That is deliberate and not a typo: the detail path is a smoothing of a level path, and the
usual reason it goes stale is that the level path underneath it is no longer walkable.
Rebuilding only the smoothing would produce a path through a wall.

The start heading handed to the detail builder is the *negated* body yaw. The body rotation
and the world direction convention disagree in sign; a rebuild that unifies the two
conventions must drop this negation, and one that keeps the original's conventions must
keep it, or every creature will begin each path turning the wrong way.

The completion test asks the detail path whether it is completed with the "patrol path"
flag inverted. A patrol-style detail path is considered complete only at its very last
point; a non-patrol one is complete as soon as the creature is near the end. The game path
pipeline always sets patrol-style, so the inversion makes the check the lenient one.

Deferral is refused when the caller demanded an at-once build, when the configuration
disables that worker stage, or when the object is being destroyed — a queued builder
holding a pointer to a dying creature is the failure this guards against.

## `show_game_path_info`

**Contract** — writes a diagnosis to the log when the coarse search fails: the creature's
name, the current level and its position on the graph, the destination level and position
if the destination vertex is valid at all, the destination vertex's four terrain-mask
bytes, and every terrain mask the creature's location manager will accept. It reads
nothing and changes nothing.

**Notes** — this is the intended debugging procedure for "a creature refuses to travel",
and it encodes what the failure almost always is: the destination's terrain mask is not in
the creature's accepted set, so the search had no admissible vertex to end on. The four
mask bytes are the game graph's per-vertex terrain classification; a rebuild that changes
their width changes the shipped graph data's meaning. Keeping this readout is worth more
than the code it costs.
