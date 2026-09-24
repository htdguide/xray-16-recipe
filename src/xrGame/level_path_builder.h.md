# src/xrGame/level_path_builder.h

> The step of a creature's path pipeline that runs the coarse level-graph search, off the main thread, with a cooldown so a creature that cannot get anywhere stops trying every frame.

**Needs** — [`movement_manager.h`](movement_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_builder.h`](detail_path_builder.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md) · [`movement_manager_level.cpp`](movement_manager_level.cpp.md) · [`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md) · [`stalker_movement_manager_obstacles.cpp`](stalker_movement_manager_obstacles.cpp.md) · [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md)
**Tier floor** — T2: schedules a graph search onto a worker and advances a state machine

## Purpose

Building a creature's path is a chain of increasingly fine searches: pick a destination on
the coarse cross-level graph, then a route across the level's navigation mesh, then a
smooth walkable curve through that route. This file is the middle step. Its two real jobs
are to get the mesh search off the frame — path searches are the AI's dominant cost and a
dozen creatures re-pathing in one frame is a visible hitch — and to stop a creature whose
destination is genuinely unreachable from re-running that search forever.

It has no implementation file; the whole thing is short enough to live in the declaration.

## State

```text
RECORD LevelPathBuilder EXTENDS DetailPathBuilder
  start_vertex     : int            # navigation vertex to search from
  dest_vertex      : int            # navigation vertex to search to
  precise_position : optional<(real, real, real)>  # exact destination within dest_vertex
  extrapolate      : bool           # whether the detail step may run past the route's end
  last_fail_time   : int            # global clock, milliseconds; 0 means never failed
  use_fail_delay   : bool = true    # false disables the cooldown entirely
```

**Invariant** — `start_vertex` and `dest_vertex` are both valid navigation vertices at the
moment they are set. This is checked on setup rather than on use, because the failure is an
authoring or logic bug in the caller and is far easier to attribute there.

**Invariant** — the precise destination is *copied* into the builder, not referenced. The
caller's value is typically a stack temporary or a position that moves; the search runs
later and on another thread, so the value must outlive the call that supplied it. This is
the file's one genuine ownership decision and a rebuild must reproduce it, however it
expresses ownership.

**Invariant** — the cooldown is two seconds of global time after a failed search. A creature
that fails to path re-enters the same pipeline state and would otherwise re-run the same
whole-graph search on the very next frame, which is the most expensive thing the AI does and
is guaranteed to fail again until the world changes. Two seconds is a tuned value with no
derivation: long enough to make repeated failure cheap, short enough that a door opening is
noticed promptly.

## `setup`

**Contract** — records the search's parameters. Does not start it. The precise destination
is optional; absent means the detail step aims at the destination vertex's own position
rather than a point inside it.

## `register_to_process`

**Contract** — marks the creature as waiting on an off-frame computation and, **unless the
cooldown is still running**, queues the search onto the engine's parallel work list.

The asymmetry is deliberate and is the subtle part: during the cooldown the creature is
still marked as waiting, but nothing is queued. It therefore stands still until the cooldown
expires and the pipeline state is re-entered, rather than spinning. A rebuild that queues
unconditionally and checks the cooldown only inside the work item burns a task slot per
frame per stuck creature.

## `process_impl`

**Contract** — the actual work, run on whichever thread drains the parallel list. Clears the
waiting flag, runs the level-graph search, and either records a failure or hands off to the
detail step. Returns nothing; all communication is through the creature's movement state.

```text
FUNCTION process_impl()
  creature.waiting_for_computation = false
  level_path.build(start_vertex, dest_vertex)

  IF level_path failed THEN
    IF use_fail_delay THEN last_fail_time = now
    creature.path_state = BuildLevelPath        # re-enter THIS step, after the cooldown
    RETURN

  level_path.select_intermediate_vertex()       # how far along the route to aim right now
  creature.path_state = BuildDetailPath

  detail.patrol_mode = extrapolate
  detail.start_position  = creature.position
  detail.start_direction = from the creature's CURRENT body yaw, negated, level
  IF precise_position IS PRESENT THEN detail.dest_position = precise_position

  hand the route and the intermediate index to the detail step
  run the detail step immediately, in this same call
```

**Invariants** — the ordering is load-bearing throughout. The waiting flag is cleared
*first*, so that a failure still releases the creature. The intermediate vertex is selected
*after* the search and *before* the detail step, because it is what bounds how much of the
route the detail step smooths — the whole route is not smoothed at once, only the portion
the creature will walk soon. The detail step's start position and direction are read from
the creature's live state at this instant, not from the search's start vertex, because the
creature has been moving while the search was queued and a curve that begins where it no
longer is produces a visible snap.

The detail step is run **inline** rather than queued again: the route is already in hand and
the smoothing is cheap relative to the search, so a second hop through the work list would
cost a frame of latency for no gain.

**Notes** — the start direction is built from the body's yaw with the sign inverted. The
inversion is a convention mismatch between the body's yaw and the direction constructor, not
a decision; a rebuild picks one convention and has no negation.

## `process`

**Contract** — the entry point the parallel work list calls. Re-checks the cooldown — the
item may have been queued before a failure landed — and otherwise asks the creature to run
the whole level-path step, which is what eventually reaches `process_impl`. The indirection
through the creature exists so that the creature can refuse or redirect the work; it is not
a plain forward.

## `remove`

**Contract** — cancels a queued search. Clears the waiting flag if it is set and removes
this builder's work item from the parallel list. Must be called before the creature is
destroyed, or the list holds an item pointing at freed state. Safe to call when nothing is
queued.

## `dest_vertex_id` · `use_delay_after_fail`

**Contract** — read the destination the builder is aiming at, and turn the failure cooldown
off. The cooldown is disabled for creatures whose destination changes constantly enough that
a stale failure would hold them still — the flag exists for exactly that case and defaults
to on.
