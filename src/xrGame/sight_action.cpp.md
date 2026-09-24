# src/xrGame/sight_action.cpp

> Turns one look order into head and torso target angles, once per sight-manager tick and again every frame for the two orders that need it.

**Needs** — [`sight_action.h`](sight_action.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`Inventory.h`](Inventory.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — reached through its declarations in [`sight_action.h`](sight_action.h.md); callers name that, not this file.
**Tier floor** — T2: angle arithmetic and a per-frame state machine; no layout or device concern

## Purpose

A creature's aiming has two layers. The *sight manager* owns the smoothing — it moves the
head and torso toward a target at a bounded angular speed. This file owns what the target
*is*. One look order names a sight type; executing it writes the head's target yaw and
pitch (and, for torso orders, indirectly the body's) and nothing else. Everything
downstream reads those two numbers.

The split matters because the two layers run at different rates. `execute` runs when the
manager schedules the action; `on_frame` runs every rendered frame. Only the orders whose
target moves *between* manager ticks — lean-out glancing, and aiming at a live enemy with
lead — implement the second. A rebuild that collapses the two rates will either jitter the
head or make aimed fire lag its target.

## State

The running state lives on the action record declared in
[`sight_action.h`](sight_action.h.md); this file is what gives it meaning. Only one sight
type's slice of it is live at a time.

```text
RECORD sight_action_running_state
  initialized          : bool          # invariant: execute/on_frame only run while true
  start_time           : int           # when the order was initialized
  internal_state       : int           # glance state machine: 0 | 1 | 2
  start_state_time     : int           # entry time of internal_state
  stop_state_time      : int           # dwell the current internal_state must serve
  cover_yaw            : real          # the yaw the cover search picked; the glance oscillates about it
  state_fire_object    : int           # aimed-fire state machine: 0 (tracking) | 1 (settled)
  state_fire_switch_time : int
  holder_start_position : vector       # the shooter's position when state 1 was entered
  object_start_position : vector       # the target's position when state 1 was entered
  already_switched     : bool          # one free re-latch per settle; see `execute_fire_object`
  vector3d             : vector        # payload: a direction, a point, or a predicted aim point
```

## `initialize`

**Contract** — marks the order running, stamps its start time, and runs the per-type entry
step for the two sight types that have one (the lean-out glance and aimed fire at an
object). Fails if the order is already running: the lifecycle is
initialize → (execute | on_frame)\* → finalize, and re-entering it would reset a running
state machine mid-flight.

**Invariants** — the aimed-fire switch time is stamped here as well as the start time, so
the settle timer in that state machine is measured from the order's own start, not from
process time zero.

## `finalize`

**Contract** — clears the running flag. Nothing else: the order's payload survives so the
manager can compare a re-issue against it (see [`sight_action_inline.h`](sight_action_inline.h.md)).

## `execute`

**Contract** — dispatches on sight type to one of ten per-type bodies. Runs at the sight
manager's rate. An unhandled sight type is a programming error, not a no-op — see the
note.

```text
FUNCTION execute()
  MATCH sight_type
    current_direction     -> head.target = head.current        # freeze where you are looking
    path_direction        -> look along the path (sight manager)
    direction             -> decompose payload vector to yaw/pitch, negate both
    position              -> aim at payload point from the eye
    object                -> aim at the tracked entity
    cover                 -> pick the least-exposed yaw from the level graph
    search                -> same search, but torso-free and pitched up
    cover_look_over       -> the glance state machine
    fire_object           -> the aimed-fire state machine
    animation_direction   -> surrender the angles to the playing animation
```

**Notes** — two sight types in the enumeration never reach this switch:
`fire_position` is rewritten to `position` at construction, and `look_over` is only ever
read by the aiming-position query in [`sight_manager.cpp`](sight_manager.cpp.md). The
enumeration is wider than the dispatch, and a rebuild should narrow it rather than
reproduce the dead arms.

## Sight-type bodies

### Freeze and path orders

**Contract** — *current direction* copies the head's current angles into its target, which
stops the head where it is without disabling the smoothing. *Path direction* delegates
entirely to the sight manager's path look, which faces the next travel point.

### `execute_direction`

**Contract** — reads the payload vector as a heading/pitch pair and negates both. The
negation is the engine's convention: the movement layer's yaw grows the opposite way from
the vector decomposition, and this is the single place the two conventions meet for a
direction order.

### `execute_position`

**Contract** — aims from a given eye point at the payload point. Routes through the torso
variant or the head-only variant depending on the order's torso-look flag; the difference
is which of the two angle solvers in
[`sight_manager_target.cpp`](sight_manager_target.cpp.md) is used, and the torso one
substitutes a forward direction when the two points coincide.

### `execute_object`

**Contract** — aims at a tracked entity. The aim point is the entity's centre *except*
that for a living creature the horizontal components are taken from its ground position
and the vertical from its centre — and the shooter's own eye point is treated the same
way.

```text
FUNCTION execute_object()
  look_at = target.center
  from    = my.eye_position
  IF target IS NOT a creature OR target IS alive
    look_at.horizontal = target.ground_position.horizontal
    from = my.center
    from.horizontal = my.eye_position.horizontal
  aim(look_at, from)                      # torso or head-only per the order's flag
  IF no_pitch THEN head.target.pitch = 0
```

**Invariants** — the horizontal-from-position, vertical-from-centre rule is what keeps a
creature aiming at a standing target's torso rather than at the bounding-box centre of a
model whose mesh is offset. A *corpse* is exempt: the raw centre is used, because a
ragdolled body's ground position is meaningless.

**Notes** — the `no_pitch` flag zeroes pitch *after* the solve rather than solving in the
horizontal plane. The difference is visible: the yaw is the yaw to the real target, so a
creature looking at something above it still turns the right way.

### `execute_cover` and `execute_search`

**Contract** — both ask the level graph for the yaw from the creature's current navigation
vertex that exposes the least of the map to a hostile — the "less cover look" search in
[`sight_manager_target.cpp`](sight_manager_target.cpp.md). The torso variant widens the
head-turn window to a half-turn; the head-only variant uses the default window. *Search*
additionally forces the torso flag off and pitches the head up by an eighth turn, which is
what makes a searching creature sweep upward rather than stare at the floor.

**Notes** — the search body still contains the torso branch even though it has just
forced the flag off, so one arm is unreachable. It is dead in the original and should not
be reproduced; what survives is that a search order is *always* head-only.

### The lean-out glance — `initialize_cover_look_over`, `execute_cover_look_over`

**Contract** — a three-state machine that makes a creature in cover repeatedly glance a
little off its cover direction and come back. Entry runs the cover search once, records
the resulting yaw as the anchor, and starts in the *settling* state.

```text
FUNCTION glance_step()
  MATCH internal_state
    2, 0 ->                                    # holding the cover yaw
      IF dwell elapsed AND head has arrived
        internal_state   = 1
        start_state_time = now
        stop_state_time  = 3500
        head.target.yaw  = cover_yaw + random in [-1/16 turn, +1/16 turn]
    1 ->                                       # holding the glance yaw
      IF dwell elapsed AND head has arrived
        re-run the cover search                # the anchor may have moved
        internal_state   = 0
        start_state_time = now
```

**Invariants** — the transition requires *both* the dwell to expire and the head to have
actually arrived at its target. Time alone is not enough: if the head is still turning,
retargeting would make it never settle and the glance would degenerate into a continuous
sweep.

State 2 and state 0 are the same behaviour and differ only in one thing: state 2 is the
first hold, and the head-speed override below is suppressed while in it. The creature
turns to its cover direction at normal speed, and only the *glances after that* are slow.

**Notes** — the dwell is 3.5 seconds in both directions and the glance amplitude is a
sixteenth of a turn either side of the anchor. Neither number is derived from anything;
they are tuned for how a person in cover reads on screen, and they are not in
configuration, which means a rebuild cannot retune them without a code change. An
out-of-range internal state hard-fails in debug builds and silently resets to state 0
otherwise — the shipping behaviour is the one to keep.

### Speed overrides

**Contract** — a look order may ask the sight manager for a non-default turn speed. Body
speed is never overridden. Head speed is overridden only by the glance order, and only
once it has left its first hold; the glance speed is a *sixteenth* of a turn per second,
roughly four times slower than a normal head turn.

**Invariants** — asking for the glance head speed while the order is not a glance order is
a programming error; the value is meaningless for any other type.

### Aimed fire at an object — `initialize_fire_object`, `execute_fire_object`, `predict_object_position`

**Contract** — the most involved of the orders, and the one that decides how an armed
creature's aim behaves. It has two states. In **tracking** (0) the aim point is recomputed
every frame with lead. In **settled** (1) the aim point is *frozen* and the creature holds
it, which is what lets it actually fire instead of chasing a moving reticle forever.

```text
FUNCTION fire_step()
  MATCH state_fire_object
    0 ->                                        # tracking
      predict_object_position(exact = false)    # aim at the centre-ish point, with lead
      IF head has NOT arrived          THEN stay
      IF no active inventory item      THEN stay
      IF cannot kill the target, or would hit a squadmate THEN stay
      IF target is nearer than 5 m     THEN stay
      state_fire_object     = 1
      state_fire_switch_time = now
      remember my position and the target's
    1 ->                                        # settled
      IF now >= state_fire_switch_time + 1500
        IF target is farther than 5 m
          IF I have moved more than 5 cm     -> re-latch, allow one more settle
          ELSE IF the target has moved 5 cm  -> re-latch, allow one more settle
        IF not already_switched              -> re-latch once on time alone
      predict_object_position(exact = true)     # aim at the visible point of the body
```

**Invariants** —

- The four tracking guards are all *reasons not to commit*: a head still turning, an
  empty hand, a shot that cannot kill or would hit a friend, and a target so close that
  precision is pointless. Passing all four is what "ready to shoot" means, and the order
  in which they are checked is irrelevant — but the set is not.
- Re-latching resets the aim point to the sight manager's current object position, not to
  a fresh prediction, which is the un-led point. The creature briefly aims *behind* a
  moving target at the moment it re-acquires; that is the visible tell of this state
  machine and it is deliberate, not a bug.
- The one-shot flag allows exactly one time-triggered re-latch per settle. Without it a
  stationary shooter facing a stationary target would re-latch every 1.5 s forever and
  never hold still.

**`predict_object_position`** — computes the aim point with lead.

```text
FUNCTION predict_object_position(use_exact_position)
  IF target is not currently visible
    aim_point = sight manager's object position      # last-known, no lead
    RETURN
  aim_point = IF use_exact_position
                the visibility point of the target   # where the raycast actually saw it
              ELSE
                the sight manager's object position
  IF the target has at least two recorded positions
    current  = newest recorded position
    previous = the newest one before it with a *different* timestamp
    IF previous is less than 300 ms old AND newer timestamp > older
      velocity = horizontal displacement / elapsed seconds
      aim_point = aim_point + velocity * aim_predict_time
  aim at aim_point from the eye
```

**Notes** —

- Lead is horizontal only. Vertical velocity is dropped, because creatures fall and jump
  and leading a fall makes a creature shoot the ground.
- The search backwards for a differently-timestamped sample exists because the position
  history can contain several entries from one frame; dividing by a zero interval would
  produce an infinite velocity. What survives is the requirement: *the two samples used
  for velocity must be from different instants.*
- The 300 ms staleness window means a target that stops being sampled is not led at all
  after a third of a second — a creature that ducks out of sight is not shot at where it
  would have been.
- The lead time is a tunable console setting defaulting to 0.40 s and is **not** scaled by
  the frame delta, so the aim point sits a fixed time ahead regardless of frame rate. A
  commented-out frame-scaling in the original shows the alternative was considered and
  rejected; frame-rate-dependent aim would make difficulty depend on hardware.
- The two aim points — "visibility point" and "object position" — differ: the first is
  where the vision raycast actually found the target, the second is the geometric aim
  solve. Tracking uses the second and settled fire uses the first, so the creature settles
  onto a point it has genuinely seen.

### `execute_animation_direction`

**Contract** — the inverse of every other order: instead of commanding angles, it *reads*
them back out of the model's transform while an animation owns the creature's movement,
and copies the body angles into the head's target so the head does not fight the clip.
When the animation is not driving movement, only the head follows the body.

**Notes** — this is the order that makes a scripted animation look right. Every other
sight type writes a target the smoothing chases; this one keeps the smoothing's state
consistent with a pose it does not control, so that when the animation ends the head does
not snap.

## `remove_links`

**Contract** — told that an entity is being destroyed. If this order was tracking that
entity, it degrades in place into a fixed-direction order pointing where the head was
already commanded to go. Never leaves a dangling reference and never leaves the creature
without an order.

**Invariants** — the degradation must preserve the *commanded* direction, not the current
one: the creature keeps turning to where it was looking rather than freezing mid-turn,
which is how a creature whose target dies finishes its turn and then re-plans.

## `target_reached`

**Contract** — true when the head's current yaw has arrived at its target yaw, compared
after normalizing both to a signed half-turn range and with a tolerance. Pitch is not
considered: arrival is a yaw question, because every dwell decision in this file is about
horizontal turning.

