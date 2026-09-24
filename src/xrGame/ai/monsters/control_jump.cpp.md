# src/xrGame/ai/monsters/control_jump.cpp

> The jump ability: a four-stage animated leap that seizes the whole body, hands the arc to the physics, lands by detecting its own deceleration, and hits whatever it passes through.

**Needs** — [`control_jump.h`](control_jump.h.md) · [`control_manager.h`](control_manager.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`monster_velocity_space.h`](monster_velocity_space.h.md) · [`trajectories.h`](../../trajectories.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`control_jump.h`](control_jump.h.md); callers name that, not this file.
**Tier floor** — T2: raycasts, a ballistic solve and per-frame steering during flight

## Purpose

The most elaborate ability in the chapter, and the template every other one follows. A jump
is four animation stages — *prepare*, *prepare in move*, *glide*, *ground* — of which a
given creature uses two, three or four; a ballistic impulse handed to the physics at the
moment the glide clip starts; an optional steering of the creature's heading while airborne
so it tracks a moving target; a landing detected from the body's own deceleration; and a
hit applied to whatever the creature passes through on the way.

Every part of it is authored. The clips come from the creature's model, the distances and
angles from its configuration section, and the stage set from which clip names the creature
supplies.

## State

```text
RECORD Jump                             # the run-time half
  anim_state_current, anim_state_prev : one of { prepare, prepare_in_move, glide, ground, none }
  time_started        : int (ms)        # when the ballistic phase began; 0 means not flying
  time_next_allowed   : int (ms)        # earliest the next jump may start
  jump_time           : real (seconds)  # the ballistic flight duration
  target_position     : vector          # where the arc is aimed
  jump_start_pos      : vector
  object_hitted       : bool            # the once-per-jump hit has been applied
  velocity_bounced    : bool            # the landing signal has been seen

RECORD JumpPayload                      # what a capturer writes
  target_object       : optional<Object>
  target_position     : vector
  force_factor        : real            # overrides the authored jump factor; negative means "unset"
  flags               : bit set, see below
  state_prepare        : { motion }
  state_prepare_in_move: { motion, gait_mask }
  state_glide          : { motion }
  state_ground         : { motion, gait_mask }
```

The flags are the ability's vocabulary and each one names a real choice:

| Flag | Meaning |
|---|---|
| `prepare_skip` | begin at the glide; no wind-up at all |
| `prepare_in_move` | wind up while running forward along a built line rather than standing |
| `glide_on_prepare_failed` | if the wind-up cannot be staged, glide anyway instead of declining |
| `glide_play_anim_once` | do not restart the glide clip when it ends; hold the last pose |
| `ground_skip` | no landing clip; the jump ends on touchdown |
| `use_target_position` | aim at the recorded position even though a target object is set |
| `dont_use_velocity_bounce` | do not subscribe to the deceleration signal |
| `use_auto_aim` | steer toward the target object while airborne |
| `enable_predict_position` | lead a moving target — declared, see Notes |

**Invariants** — `time_started` of zero means the ballistic phase has not begun, and the
landing test short-circuits on it. The hit is applied at most once per jump.

**Notes** — position prediction is declared, flagged and plumbed through, and the function
that would do it returns its argument unchanged. Leading a moving target was intended and
never implemented; the flag is inert.

## Authored parameters

Read from the creature's configuration section by `load`. All required except the last:

| Key | Meaning |
|---|---|
| `jump_delay` | milliseconds before another jump is allowed |
| `jump_factor` | divides the physically minimal flight time — a larger factor means a flatter, faster arc |
| `jump_ground_trace_range` | how far below the creature's centre a ray must find ground to count as landed |
| `jump_hit_trace_range` | how far ahead the in-flight hit ray reaches |
| `jump_build_line_distance` | how long the landing run-out line is |
| `jump_min_distance`, `jump_max_distance` | the distance band a jump may cross |
| `jump_max_angle` | the heading error beyond which the creature will not jump |
| `jump_max_height` | the vertical difference beyond which the target counts as a different floor |
| `jump_auto_aim_factor` | optional, default zero; scales the in-flight steering authority |

## `check_start_conditions`

**Contract** — refuses if the ability is already running, or if any of the four body
resources is held by another ability. No distance or angle test here; those are in
`can_jump`, which the creature's custom manager calls separately.

## `activate`

**Contract** — seize all four body resources, subscribe to animation start, animation end
and — unless suppressed — the deceleration signal, then begin the jump aimed either at the
target object's root bone or at the recorded position.

**Notes** — the deceleration subscription clears its own suppression flag as it subscribes,
so the flag is one-shot: a caller that suppressed the signal for one jump gets it back for
the next.

## `start_jump`

**Contract** — choose the opening stage and start it. Three cases, in order.

```text
FUNCTION start_jump(point)
  clear the hit and bounce flags; record the start position and target
  tell the creature to ignore collision damage until landing

  IF flag prepare_skip THEN
    stage <- glide, previous <- prepare      # the pair that means "the arc starts now"
    stop the path and the movement
  ELSE
    prepared <- false
    IF flag prepare_in_move THEN
      distance <- (duration of the wind-up clip) * (speed of its gait)
      point_ahead <- my position + my facing * distance
      IF point_ahead is accessible AND the mesh admits a straight walk to it THEN
        IF a line path to it can be built with (that gait OR stand) THEN
          enable the path, LOCK the path channel, stop turning
          stage <- prepare_in_move
          prepared <- true
    IF NOT prepared THEN
      IF a standing wind-up clip exists THEN
        stage <- prepare; stop the path and the movement
      ELSE
        REQUIRE flag glide_on_prepare_failed
        stage <- glide, previous <- prepare

  select_next_anim_state()
```

**Notes** — the wind-up-in-move distance is computed from the clip, not authored: the
creature runs exactly as far as the clip lasts, so the clip ends the instant the creature
reaches the launch point. That derivation recurs in every lunging ability in the chapter.

The pair (current = glide, previous = prepare) is not a state; it is the encoding that
means "the next animation-start event is the launch". See the state machine below.

## `select_next_anim_state`

**Contract** — start the clip for the current stage and advance the stage. The advance is
where the state machine actually lives, and it is small enough to state whole.

```text
FUNCTION select_next_anim_state()
  IF stage IS none THEN stop(); RETURN
  IF stage IS glide AND previous IS glide AND flag glide_play_anim_once THEN RETURN

  mark the animation payload stale and set its motion from the stage
  previous <- stage
  IF stage IS NOT glide THEN
    IF stage IS prepare THEN stage <- glide     # prepare jumps straight to glide
    ELSE                     stage <- next stage in order
  # a glide stage leaves `stage` at glide: the glide repeats until something stops it

  IF steering is in force THEN
    hand the physics character an air-control authority of 100 * auto_aim_factor
```

**Notes** — the ordering of the stage enumeration is load-bearing: *prepare*,
*prepare in move*, *glide*, *ground*. "Advance to the next stage" is an increment over that
order, with one special case, and the glide is a fixed point. A rebuild that reorders the
enumeration breaks the ability silently.

The factor of one hundred converting the authored steering value into the physics layer's
air-control authority is a unit conversion with no recorded derivation.

## `on_event`

**Contract** — three events drive the whole jump.

- **animation start.** When the glide clip starts for the *second* time — that is, when the
  current and previous stages are both glide — this is the launch. The physical flight time
  is solved, the physics is asked to jump to the target in that time, the start timestamp
  and the next-allowed timestamp are set, the heading target is aimed at the target (unless
  steering will do it) with a turn rate of exactly the heading error divided by the flight
  time, linear dependency is switched off, and the glide clip's playback rate is stretched
  so the clip lasts exactly as long as the flight. Any other animation start resets the
  playback rate to as-authored.
- **animation end.** Advance the stage.
- **deceleration signal.** A *negative* ratio — the body slowed abruptly — while flying and
  not already bounced means impact: if a ray finds ground below, this is a landing and the
  run-out begins; otherwise the jump is abandoned.

**Notes** — "the glide clip starting for the second time" is the launch trigger, and it is
the single most obscure decision in the chapter. Reading it plainly: the first glide entry
sets the clip; the clip starts; the handler sees current = glide, previous = glide (because
the advance left it there) and fires. A rebuild should make this an explicit "on launch"
step rather than an encoding.

Three numbers are made to agree at launch and a rebuild must reproduce all three or the
jump reads wrong: the physics flight time, the clip's playback rate, and the turn rate.
The creature turns to face its target exactly as it lands, and its glide clip finishes at
the same moment.

## `update_frame`

**Contract** — per frame while airborne: finish when the landing signal has been seen and
the run-out is complete; steer if steering is in force; test for a hit; adopt the path's
speed while on a built line; and check for touchdown.

```text
FUNCTION update_frame()
  IF velocity_bounced AND the path is within a tenth of a unit of its end THEN
    stop(); RETURN

  IF stage IS glide AND steering is in force THEN
    heading target <- angle to the target object
    heading rate   <- (heading error) / jump_time
    linear dependency <- off

  hit_test()

  IF moving on a path THEN
    movement target <- the speed the current waypoint calls for, instantly

  IF is_on_the_ground() THEN grounding()
```

## `is_on_the_ground`

**Contract** — has the creature landed. Answers no before launch and no before the solved
flight time has elapsed; then casts a ray straight down from the creature's centre against
static geometry and answers yes if ground is found within the authored trace range.

**Notes** — the time gate matters as much as the ray: without it, a creature jumping from a
low ledge would find ground beneath it throughout the arc and land immediately.

## `grounding`

**Contract** — begin the landing run-out. If the creature has no landing clip, no landing
gait, or the run-out is suppressed, the jump simply ends. Otherwise a straight line is
built the authored run-out distance ahead along the creature's facing, restricted to the
landing gait or standing; the path is enabled and **locked**, turning is stopped, the
ballistic timers are cleared, and the landing clip starts. A line that cannot be built ends
the jump.

**Notes** — locking the path is what stops the ordinary path builder from immediately
replanning over the run-out. Every ability that builds its own line does this.

## `hit_test`

**Contract** — apply the jump's damage at most once, to the target object, when the creature
is close enough and pointed at it. Two tests, and the second overrides the first.

```text
FUNCTION hit_test()
  IF already hit OR no target object THEN RETURN

  ray forward from my centre against objects, to the authored hit range
  IF the ray struck the target object within range THEN hit <- true

  IF NOT hit THEN
    hit <- true                              # provisionally
    IF distance to the target > hit range THEN hit <- false
    IF the target's heading is outside +/- a twelfth of a turn of my facing THEN hit <- false
    IF the target's pitch   is outside +/- a twelfth of a turn of my facing THEN hit <- false

  IF hit THEN apply the creature's in-jump hit to the target
```

**Notes** — the second test exists because the ray misses far more often than it should:
the creature and its target are both moving and the ray is a line, so a claw that visibly
connects frequently traces past. The cone test is the forgiving fallback and it is what
actually lands most jump attacks. A rebuild that uses a swept volume can drop it.

The cone is a twelfth of a turn either side in both heading and pitch — thirty degrees
total in each axis — and is a constant here.

## `can_jump`

**Contract** — may this jump be attempted at a target. Six tests, all authored:

```text
FUNCTION can_jump(target, aggressive) -> bool
  IF a cooldown is pending THEN
    # an aggressive jump may cut the remaining delay by two thirds
    IF next_allowed - (aggressive ? 2*jump_delay/3 : 0) > now THEN RETURN false
  IF the target is outside my restrictions THEN RETURN false
  distance <- my position to the target
  min <- aggressive ? min(1, jump_min_distance) : jump_min_distance
  IF distance < min OR distance > jump_max_distance THEN RETURN false
  IF my heading error to the target > jump_max_angle THEN RETURN false
  IF the vertical difference > jump_max_height THEN RETURN false
  IF a wind-up is required THEN
    verify the required clips exist; for a moving wind-up, also verify
    the mesh admits the run-up, and fall back to the standing wind-up if it does not
  RETURN true
```

**Notes** — the *aggressive* variant is a creature-authored mood, not a parameter: it cuts
the cooldown to a third and drops the minimum distance to one world unit, so an aggressive
creature can jump repeatedly and from almost on top of its target. Which creatures may be
aggressive is decided by the creature class.

The maximum height test is described in the source as "is the target on the same floor",
which is exactly right: it is a cheap substitute for asking whether the arc is navigable.

## `jump_intersect_geometry`

**Contract** — would the ballistic arc to a target pass through geometry. Solves the flight
time and the launch velocity, then sweeps a box along the trajectory against the collision
database. Both endpoints are raised by 1.2 world units and the target end is pulled back by
one unit along the approach direction; the swept box is 0.8 by 1.4 by 0.8. A target closer
than one world unit is never considered obstructed.

**Notes** — all five numbers are constants here. The raise and the pull-back together make
the test ignore the floor at both ends and the target's own body, which is the point: a
jump that ends *at* a creature must not be rejected for intersecting it. The box is a crude
stand-in for the jumper's own bulk. A debug build keeps the sweep's picks and the triangles
it struck on the creature for on-screen inspection.

## `calculate_jump_time`

**Contract** — the flight duration: the physically minimal time to reach the target,
divided by a factor. The factor is the caller-supplied force factor when one is set and
positive, otherwise the creature's authored jump factor.

**Notes** — dividing the *minimal* time by a factor greater than one produces an arc flatter
and faster than a ballistic minimum, which is how these creatures cross distances that
would need an implausible lob. It is the one place the chapter deliberately leaves physical
plausibility behind, and the factor is per-creature authored.

## `in_auto_aim`, `relative_time`, `stop`, `remove_links`, `on_release`

**Contract** — `in_auto_aim` is the conjunction of: a target object exists, the steering
flag is set, the authored steering factor is non-zero, and the creature is in the glide
stage. `relative_time` is the fraction of the flight elapsed, clamped to one — the number a
creature's own code uses to drive anything time-varying during the arc. `stop` raises the
jump-end event and nothing else; ending the ability is the custom manager's job.
`remove_links` clears the target object when that object is destroyed.

`on_release` is the mirror of `activate`: unlock the path, restore linear dependency on the
direction, release all four resources, unsubscribe from the three events, clear the physics
air-control authority, reset the path builder's parameters, and let the creature take
collision damage again.

**Notes** — restoring linear dependency and collision damage in `on_release` rather than
where they were disabled is what makes the ability safe to abort mid-flight. Every abort
path in this file reaches `on_release` through the manager.
