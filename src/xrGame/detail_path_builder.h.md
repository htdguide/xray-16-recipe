# src/xrGame/detail_path_builder.h

> Runs a detail-path build on the engine's parallel work queue and reports the outcome back into the movement manager's state machine.

**Needs** — [`movement_manager.h`](movement_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`level_path_builder.h`](level_path_builder.h.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md) · [`movement_manager_level.cpp`](movement_manager_level.cpp.md) · [`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md)
**Tier floor** — T2: scheduling a unit of work and a state transition

## Purpose

Building a detail path is expensive — it searches turning-circle trajectories between every
pair of enabled velocities at every corner of the coarse path — and a level can have dozens
of creatures wanting one in the same frame. This small adapter is what lets the work happen
off the main update: it registers a callable with the engine's parallel work list, and on
completion advances the owning movement manager to its next path state.

It is header-only and has no implementation file; the whole thing is eight short members.

## State

```text
RECORD DetailPathBuilder
  object              : MovementManager    # required; the owner
  level_path          : list<int>          # borrowed, not owned: the coarse vertex list
  path_vertex_index   : int                # where in that list to start
```

**Invariant** — the coarse path is held **by reference**. It belongs to the movement
manager's level-path stage and must outlive the build. Since the movement manager is also
what waits on the build, this holds — but it is the reason a build cannot be cancelled by
dropping it, only by the explicit removal below.

## `setup`

**Contract** — points the builder at a coarse path and a starting index. Does not schedule
anything.

## `register_to_process`

**Contract** — marks the owner as waiting for a distributed computation and appends the
build to the engine's parallel work list for this frame. Returns immediately.

**Invariants** — the waiting flag is raised *before* the work is queued. The movement
manager's update checks that flag to decide whether it may proceed, so raising it after
queueing would leave a window in which the manager runs its next stage against a path that
is being rewritten.

## `process` · `process_impl`

**Contract** — the work itself, run on a worker. Clears the waiting flag, builds the detail
path, notifies the owner, and then moves the owner's path state machine to one of two
places:

```text
FUNCTION process_impl(separate_computing)
  IF separate_computing THEN clear the owner's waiting flag
  build the detail path from the coarse path and the start index
  notify the owner that a path was built
  IF the detail path failed THEN
    owner.path_state = build_level_path      # go back a stage and try a new coarse path
  ELSE
    owner.path_state = path_verification     # go forward and check it is still walkable
```

**Invariants** — a failed detail path sends the movement manager **back to the coarse
stage**, not to a failure state. This is the engine's whole response to "I cannot smooth
this corner": find a different sequence of navigation vertices. Without that loop a creature
that cannot turn tightly enough for one doorway would simply stop.

**Notes** — the `separate_computing` parameter distinguishes the queued call from a direct,
synchronous call made on the main thread by a caller that cannot wait a frame. Only the
queued path clears the waiting flag, because the direct path never raised it.

## `remove`

**Contract** — cancels a queued build: clears the waiting flag if it is set and withdraws the
callable from the parallel work list. Must run before the owner is destroyed, since the work
list holds a bound reference to it.

**Notes** — the callable is identified for removal by rebuilding an equal bound-method value,
which is a C++ mechanism for "the same job". A rebuild wants a handle returned at
registration.
