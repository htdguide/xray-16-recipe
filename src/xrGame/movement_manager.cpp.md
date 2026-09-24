# src/xrGame/movement_manager.cpp

> The path pipeline's lifecycle and state driver: what invalidates a path, which stage runs next, and where a moving creature will be a moment from now.

**Needs** — [`movement_manager.h`](movement_manager.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`game_location_selector.h`](game_location_selector.h.md) · [`level_location_selector.h`](level_location_selector.h.md) · [`game_path_manager.h`](game_path_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`patrol_path_manager.h`](patrol_path_manager.h.md) · [`location_manager.h`](location_manager.h.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`detail_path_builder.h`](detail_path_builder.h.md) · [`steering_behaviour.h`](steering_behaviour.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`mt_config.h`](mt_config.h.md) · [`Level.h`](Level.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`movement_manager.h`](movement_manager.h.md); callers name that, not this file.
**Tier floor** — T2: a state machine over four search stages, with a hot prediction walk

## Purpose

Every walking creature owns one of these. Its job is to turn *"be at that place"* into *"this
frame, step here"*, and it does so through a pipeline of up to four stages, each producing
the input to the next:

```text
game location selector  →  which coarse graph vertex leads toward the goal
game path manager       →  the route across the coarse graph
level path manager      →  the route across this level's navigation mesh
detail path manager     →  a smoothed, followable curve of travel points
```

Which prefix of that pipeline runs is the **path type**, and it is the one choice a caller
makes. This file owns the state machine over those stages; the four siblings named in
[`movement_manager.h`](movement_manager.h.md) own the stages themselves.

Two ideas here are worth more than the rest.

**Actuality, not dirtiness.** Every stage reports whether its result is still *actual*. The
manager's own actuality flag is the conjunction of its own and the stages'. Anything that
could change a route — a new destination, a restrictor change, the creature having been
moved while movement was suspended — clears it, and the next update rebuilds only from the
stage the path type says to restart at. There is no invalidation graph; there is one flag
and a switch.

**Expensive stages may be deferred to a worker.** The level search and the detail smoothing
are the two costly ones, and each has a *builder* that can run it off the frame. A creature
waiting on a deferred build does no path work at all that frame. That is a correctness
constraint as much as a performance one: nothing may read a half-built path.

## State

```text
RECORD MovementManager
  object            : CustomMonster
  path_type         : {no_path, game_path, level_path, patrol_path}
  path_state        : the pipeline stage to run next
  path_actuality    : bool          # this manager's own freshness flag
  enabled           : bool          # movement suspended without losing the path
  build_at_once     : bool          # this rebuild may not be deferred to a worker
  extrapolate_path  : bool          # the detail path may continue past the last vertex
  speed             : real          # achieved
  old_desirable_speed : real        # commanded, as of the previous frame
  wait_for_distributed_computation : bool
  on_disable_object_position : (real, real, real)   # where we were when movement was suspended
  # eleven owned subordinates: two cost evaluators, four stage managers,
  # the restrictor set, the location manager, two builders
```

**Invariants**
- `path_type` is never the dummy value when read. The dummy exists only as an
  uninitialized marker and reading it is an error, not a case.
- A creature waiting on a distributed computation runs no pipeline stage.
- The detail path is the only output anything outside this component reads. The three
  upstream paths are intermediate.
- `speed` is what was achieved; `old_desirable_speed` is what was asked for last frame.
  Prediction uses the second, not the first — see `prediction_speed` below.

## `Load` / `reinit` / `reload`

**Contract** — `Load` constructs all eleven subordinates and reads the creature's navigation
tuning. Construction order matters in one place only: the restrictor set and the location
manager exist before the four stage managers, because each stage manager is handed the
restrictor set at construction and consults it on every expansion.

`reinit` resets to "standing still, no path", zeroes both speeds, re-enables movement, and
re-points each stage at its graph — the coarse graph for the game stages, the level's
navigation mesh for the level stage. It also hands the game path's output buffer to the game
location selector, which is the one direct wiring between two stages rather than through the
manager.

**Notes** — path building is explicitly *not* deferred to a worker during initialization.
The first path after a spawn or a level load is built inline, because the frame that follows
a spawn is not a frame anyone is watching and a deferred first path leaves a creature
standing still visibly.

## `update_path`

**Contract** — advances the pipeline by whatever it can do this frame. Does nothing when
movement is suspended or a worker is still building. Profiled. Two halves: decide where to
restart from, then run the stage.

```text
FUNCTION update_path()
  IF NOT enabled OR waiting on a worker THEN RETURN

  IF the cost evaluators are unset THEN install the defaults
  IF the restrictor set is not actual THEN path_actuality = false
  mark the restrictor set actual

  IF NOT actual
    make every stage inactual
    path_state = the first stage of this path type
    IF path type IS level_path
      # the destination may have become unreachable since it was chosen
      IF the destination vertex is no longer permitted
        move the destination to the nearest permitted vertex
        move the detail destination to that vertex's position
      ELSE IF the detail destination itself is no longer permitted
        snap the detail destination back onto the destination vertex
    path_actuality = true

  run the stage for this path type
  build_at_once = false
```

**Invariants** — the restrictor set's own actuality is folded in and then **reset**, every
frame, whether or not it was stale. The restrictor set is the one subordinate that is
shared with other components (the memory manager, the brain), so it cannot be left holding
a stale flag for a consumer that has not looked yet.

**Notes** — the destination repair in the level case is the only place in the pipeline
where a *goal* is rewritten rather than a path. A creature told to walk to a place that has
since been forbidden walks to the nearest place it may stand instead of failing. That is a
gameplay decision, not an error recovery: scripts routinely move restrictors under creatures
that are already walking, and the alternative — the creature stopping — reads as a bug.

Note that the repair is level-path only. A game path whose destination has become
inaccessible is not repaired, because the coarse graph has no notion of restrictors.

## `on_frame`

**Contract** — the per-frame entry point. Advances the pipeline, then consumes the detail
path against the physics character. Takes the frame delta from the creature's own client
clock rather than the device's, so a creature updated at a reduced rate by the scheduler
moves the right distance.

```text
FUNCTION on_frame(movement_control, out dest_position)
  IF enabled AND path_state IS NOT (verification OR completed)
    update_path()
  move_along_path(movement_control, dest_position, object.client_frame_delta)
```

**Notes** — the two excluded states are terminal for one cycle. *Verification* means the
detail path is being checked and must not be rebuilt under the check; *completed* means
there is nothing left to build. Skipping the update in those two is what stops a creature
standing on its destination from re-running the whole pipeline every frame.

## `actual_all`

**Contract** — is the entire path fresh, all the way down. Conjunction of this manager's own
flag and the flags of exactly the stages this path type uses. A no-path creature still has a
detail path — the local steering — so even that case has something to check.

## `enable_movement`

**Contract** — suspends or resumes motion. The subtlety is the resume.

```text
FUNCTION enable_movement(on)
  IF turning off AND was on
    remember the object's current position
  ELSE IF turning on AND was off AND the object has since moved
    path_actuality = false      # the path no longer starts where we are
  enabled = on
```

**Notes** — a creature whose movement is suspended may still be *moved* — by a physics
impulse, a script teleport, a vehicle. The remembered position is how the resume detects it.
Without this, the creature would resume following a path whose first travel point is
somewhere it no longer is, and walk backwards to reach it.

## `set_game_dest_vertex` / `set_level_dest_vertex`

**Contract** — name a destination on the coarse or the fine graph. Each folds the
corresponding stage's resulting actuality into the manager's flag, so setting a destination
that is already the current one costs nothing. The level form asserts the destination is
accessible: choosing an inaccessible destination is a caller error, distinct from a
destination that *becomes* inaccessible, which `update_path` repairs.

## `on_restrictions_change`

**Contract** — the creature's permitted space has changed. Drops actuality, cancels both
deferred builders, and tells the level stage — which caches per-vertex accessibility and must
discard it. The two upstream stages are not told: the coarse graph has no restrictors, and
the detail path is rebuilt from the level path anyway.

## `predict_position`

**Contract** — where this creature will be after a given time, walking its current detail
path at a given speed. Pure; advances a caller-held travel-point cursor as a side effect so
repeated calls walk forward cheaply. Returns the current position when there is no path, and
the path's end when the distance runs past it.

```text
FUNCTION predict_position(dt, start, inout cursor, velocity) -> position
  IF path is empty THEN RETURN start
  budget = velocity * dt
  IF cursor IS the last point THEN RETURN path.last.position

  # first leg is measured from the caller's actual position, not from the
  # cursor's point — the creature is somewhere between two travel points
  leg = distance(start, path[cursor + 1].position)
  IF leg >= budget
    RETURN start moved budget toward path[cursor + 1].position
  budget = budget - leg
  cursor = cursor + 1

  WHILE cursor < last
    leg = distance(path[cursor].position, path[cursor + 1].position)
    IF leg > budget THEN BREAK
    budget = budget - leg
    cursor = cursor + 1

  IF cursor IS the last point THEN RETURN path.last.position
  RETURN path[cursor].position moved budget toward path[cursor + 1].position
```

**Invariants** — the first leg is special-cased and measured from the *caller's* start
position. Every subsequent leg is point-to-point. Getting that wrong produces a prediction
that jumps by up to one travel-point spacing, which at aiming distances is the difference
between a hit and a miss.

Two degenerate guards: a zero-length leg returns the endpoint rather than normalizing a zero
direction, in both the first-leg and the final-leg cases.

**Notes** — this is the function everything that aims at a moving creature calls, and it is
called several times per shooter per frame. It is the hottest thing in this file, which is
why it takes the cursor by reference — a caller sweeping several time offsets walks the path
once.

The convenience form predicts from the creature's own position at
**`prediction_speed`, which is the *previously commanded* speed, not the achieved one**.
That is deliberate: the achieved speed lags the command by the physics character's
acceleration, so predicting with it consistently under-leads a creature that has just
started moving. Using last frame's command instead assumes the creature will reach what it
was asked for, which is right more often than it is wrong.

## `distance_to_destination_greater`

**Contract** — is more than a given distance left along the path. Walks travel points from
the current one, accumulating, and stops early as soon as the threshold is passed. Reports
*true* for a path too short to walk or one already completed, which is the conservative
answer for every caller — they all use it as "am I still far away".

**Notes** — early exit is the point. The question is asked with small thresholds by creatures
on long paths, and accumulating the whole remaining length would be wasted work.

## `target_position`

**Contract** — where this path ends. Returns the creature's own position when there is no
path. Reads the *last patrol point* of the detail path rather than its last travel point:
the detail path may extrapolate past the final navigation vertex (see `extrapolate_path`),
and the extrapolated tail is a smoothing artifact, not a destination.

## `teleport`

**Contract** — moves the creature to a coarse-graph vertex by emitting an authoritative
teleport event carrying the target's coarse vertex, its level vertex and its world position.
Sent reliably and ordered. The client does not move the creature itself — cross-level
movement is the alife simulation's business, and a client that moved itself would diverge
from the record that decides where it exists.

## `clear_path` / `build_level_path` / `net_Spawn` / `net_Destroy`

**Contract** — `clear_path` empties the detail path and cancels its builder. `build_level_path`
runs the level search inline, which is what the builder calls when deferral is not allowed.
`net_Spawn` forwards to the restrictor set, which is the only subordinate with spawn state.
`net_Destroy` cancels both builders **before** the restrictor set is torn down — a builder
still queued would otherwise run against a released restrictor set, which is the same
teardown-ordering rule the whole engine asserts.

## `can_use_distributed_computations`

**Contract** — may this stage be handed to a worker. Three conditions, all necessary: this
rebuild was not marked build-at-once, the corresponding build option is enabled in the
process-wide worker configuration, and the creature is not being destroyed. The third is the
one that matters — a deferred build for an object already on its way out has nowhere to
deliver.

## `on_travel_point_change` / `create_restricted_object` / `level_path_path` / `path`

**Contract** — `on_travel_point_change` relays to the detail stage, which uses it to fire
animation and sound events at path corners. `create_restricted_object` is the factory hook
letting a creature kind supply its own restrictor semantics; the default supplies the standard
one. `level_path_path` and `path` expose the level and detail paths to the stages and to
callers respectively.

## `verify_detail_path`

**Contract** — compiled out. When enabled, it would re-check the next fifteen metres of the
detail path against the restrictor set each frame and invalidate the path on the first
forbidden point, skipping the check entirely when the creature has any permitted-region
restrictor.

**Notes** — the fifteen-metre horizon and the whole feature are disabled in the shipped
build. The skip-when-permitted-regions-exist condition is the interesting half: with a
permitted region in force, the path is already constrained to it by construction, so the
verification would be pure cost. Unrecovered: why the feature is off at all. The cost is one
short walk per creature per frame, which is not obviously prohibitive, so it was probably
disabled for a behavioural reason rather than a performance one.
