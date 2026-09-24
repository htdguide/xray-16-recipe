# src/xrGame/ai/monsters/control_path_builder_base_update.cpp

> The per-frame pass: classify the state of the current path, decide whether the target needs re-resolving, and publish the result into the channel payload.

**Needs** — [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: runs every frame for every live creature

## Purpose

Three steps in a fixed order, once per frame: work out what state the path is in, re-resolve
the target if that state calls for it, and copy everything into the channel payload for the
path resource to apply on its next scheduled tick. The ordering matters — the state
classification is what the resolution decision reads, and the payload write must see the
resolution's result.

## State

Reads and writes the record in
[`control_path_builder_base.h`](control_path_builder_base.h.md).

## `update_frame`

**Contract** — the three steps. Wrapped in a named profiler zone, which is how this
chapter's cost is attributed in a capture.

```text
FUNCTION update_frame()
  update_path_builder_state()
  update_target_point()
  set_path_builder_params()
```

## `update_path_builder_state`

**Contract** — classify the path into a bit set. Exactly one of *valid*, *end* and *no
path* is chosen, the *waiting* bit may be added to the first two, and an observed failure
replaces the lot.

```text
FUNCTION update_path_builder_state()
  state <- path_valid
  IF the detailed path is empty THEN state <- no_path
  ELSE IF path_end THEN              state <- path_end

  # we asked for a path more recently than the stack last reported one,
  # or the path is stale and was built before we asked: an answer is still coming
  IF last_time_target_set > time_path_updated_external
     OR (the detailed path is not up to date
         AND its build time < last_time_target_set) THEN
    state <- state OR wait_new_path

  IF failed THEN
    state <- state OR path_failed
    state <- state WITHOUT path_valid WITHOUT wait_new_path
    failed <- false                      # consume the observation
    time_global_failed_started <- now     # open the three-second window
```

**Notes** — the *waiting* bit is the element's answer to asynchrony. The navigation stack
may spread a search over several frames, so between asking and being answered the path on
the ground is the *old* one and must not be judged. Without this bit the element would see
a stale path, call it failed, and ask again — forever.

A failure is consumed here, not when it is observed: the flag is set by the event handler
and cleared by this pass, so exactly one pass sees each failure. What outlives it is the
three-second window.

## `update_target_point`

**Contract** — re-resolve the target when the state calls for it, remember whether the
resolution landed on the same vertex as before, and mark the target current.

```text
FUNCTION update_target_point()
  reset_actuality <- false
  IF movement disabled THEN RETURN
  IF path type IS NOT level THEN RETURN      # game and patrol paths resolve elsewhere
  IF NOT target_point_need_update() THEN RETURN

  previous <- target_found
  IF global_failed() THEN find_target_point_failed()   # inside the giving-up window
  ELSE                     find_target_point_set()

  IF target_found.node == previous.node THEN
    # the level path search would consider its result still valid and skip the replan,
    # so force it to treat the path as stale
    reset_actuality <- true

  last_time_target_set <- now
  target_actual        <- true
```

**Notes** — the force-stale flag is the subtle part. Resolution can legitimately return the
same vertex it returned last time while the *position* within that vertex's cell has
changed, and the level path search keys its validity on the vertex alone. Without the
flag, the creature would keep walking to the old position inside the right cell. The flag
reaches the path resource through the payload and makes it toggle movement off and on,
which is how a replan is forced — an indirect mechanism a rebuild should replace with an
explicit invalidate.

Only the level path resolves here. A game path is aimed at a graph vertex by the alife
layer and a patrol path is a fixed authored route; neither has a target to correct.

## `set_path_builder_params`

**Contract** — copy the whole held parameter set into the channel payload: the found
target's position and vertex, the enable flag, the path type and game-graph target, the
arrival orientation, the minimum-time and extrapolation preferences, the two gait masks,
and the force-stale flag. Does nothing at all if this element is not the channel's current
capturer.

**Notes** — the silent no-op is how the whole base-element pattern works. An ability that
has seized the path leaves this element running and writing nothing, so when the ability
releases, the element resumes with its held state intact and no re-initialization.
