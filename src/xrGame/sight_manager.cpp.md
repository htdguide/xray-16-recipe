# src/xrGame/sight_manager.cpp

> Advances a creature's head and torso toward the angles its look order asked for, decides when it must turn its feet, and produces the additive bone rotations the animation layer lays over the playing clip.

**Needs** — [`sight_manager.h`](sight_manager.h.md) · [`sight_action.h`](sight_action.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`aimers_weapon.h`](aimers_weapon.h.md) · [`aimers_bone.h`](aimers_bone.h.md) · [`Weapon.h`](Weapon.h.md)
**Used by** — reached through its declarations in [`sight_manager.h`](sight_manager.h.md); callers name that, not this file.
**Tier floor** — T2: quaternion interpolation per creature per frame; hot but not device-facing

## Purpose

Aiming in this engine is *not* a bone constraint applied to a pose. It is two numbers —
the head's yaw and pitch — chased at a bounded angular speed, plus a separate correction
solved against the currently playing animation so that the weapon's muzzle actually points
where those numbers say. This file owns both, and the ordering between them is the part a
rebuild gets wrong: the angles are advanced first, the creature's world orientation is
written from the *body* angles last, and the bone correction is computed in between from
the difference between head and body.

The file splits from [`sight_manager_target.cpp`](sight_manager_target.cpp.md) along a
clean line: that file computes *what angle to want*, this one computes *how to get there*.

## State

Declared in [`sight_manager.h`](sight_manager.h.md). The module-level tuning is here:

```text
CONSTANT free_factors    = { head 0.25, shoulder 0.25, spine 0.50 }
CONSTANT danger_factors  = { head 0.00, shoulder 0.50, spine 0.50 }
CONSTANT factor_lerp_speed = 1.0            # per second; a full factor swap takes ~1 s
CONSTANT aim_min_speed   = 0.24 rad/s       # console-tunable
CONSTANT aim_min_angle   = 1/32 turn        # below this, turn at min speed
CONSTANT aim_max_angle   = 1/8 turn         # above this, turn at full speed
```

**Invariants** — the three factors in each set are how the head/body angle difference is
*distributed* across the spine, shoulder and neck. They need not sum to one and do not:
the free set sums to one, the danger set to one as well, but the danger set gives the neck
*nothing*. That is the whole point — a creature looking down its sights turns its torso and
keeps its head aligned with its weapon; a relaxed creature cranes its neck. The two sets
are crossfaded over about a second when the torso-look flag changes, which is why a
creature raising its weapon visibly settles its shoulders.

## `Exec_Look`

**Contract** — the per-frame half. Runs unconditionally whether or not aiming is enabled;
the enabled flag only gates the bone-rotation stage. Mutates the creature's body and head
angle pairs in place and, unless an animation owns the creature's movement, rewrites the
creature's world transform to face the body yaw. Does not allocate.

```text
FUNCTION exec_look(time_delta)
  IF an animation owns movement THEN body.target = body.current   # do not fight the clip
  normalize body.current, body.target, head.current, head.target to signed half-turn
  body_speed = order.body_speed IF order overrides it ELSE body.speed
  head_speed = order.head_speed IF order overrides it ELSE head.speed
  validate_angle_dependency(body.current.yaw, body.target.yaw, head.current.yaw)
  FOR each of body.yaw, body.pitch, head.yaw, head.pitch
    advance current toward target at select_speed(remaining angle, that speed), time_delta
  IF enabled
    compute_aiming(time_delta, head_speed)
    order.on_frame()
  IF an animation owns movement THEN RETURN
  rebuild the creature's world transform as a pure yaw of -body.current.yaw
```

**Invariants** —

- Normalization happens **before** the speed selection, because the remaining angle is
  measured between the normalized values; skipping it makes a creature take the long way
  around after its yaw accumulates past a full turn.
- The world transform is rebuilt from the body yaw only — pitch and roll are discarded and
  the position column is zeroed in the rotation basis. A creature's *feet* never pitch; a
  rebuild that writes the full orientation will make creatures lean.
- `on_frame` on the order runs **after** the angles have been advanced, so an order that
  reads "has the head arrived" (the glance and aimed-fire state machines in
  [`sight_action.cpp`](sight_action.cpp.md)) sees this frame's position, not last frame's.

### `select_speed`

**Contract** — scales the commanded angular speed down as the remaining angle shrinks, so
the head decelerates into its target instead of stopping dead. Below a thirty-second of a
turn the speed is the floor; above an eighth of a turn it is the full commanded speed; in
between it is linear.

```text
FUNCTION select_speed(distance, speed, min_speed, min_distance, max_distance) -> real
  IF speed <= min_speed            RETURN speed      # already slower than the floor
  IF distance <  min_distance      RETURN min_speed
  IF distance >= max_distance      RETURN speed
  factor = (distance - min_distance) / (max_distance - min_distance)
  RETURN min_speed + factor * (speed - min_speed)
```

**Notes** — this is a debug-toggleable behaviour in the original; with it off the head
turns at constant speed and snaps. The smooth path is the shipping one. The floor exists
because a purely proportional deceleration never arrives, and the arrival test in
[`sight_action.cpp`](sight_action.cpp.md) is what several state machines wait on.

### `vfValidateAngleDependency`

**Contract** — given the body's current yaw, the body's *target* yaw and the head's
current yaw, forces the body target to the head's yaw when the two would require turning
in opposite directions through more than a half turn.

```text
FUNCTION validate_angle_dependency(reference, target, other)
  a = signed angle from reference to target
  b = signed angle from reference to other
  IF a and b have the same sign              RETURN     # turning the same way, fine
  IF |a| + |b| <= half turn                  RETURN     # they meet the short way, fine
  target = other
```

**Invariants** — the pathology this prevents is the body and the head unwinding in
opposite directions past each other, which twists the spine through an impossible angle
for a frame and reads as the model briefly snapping. Collapsing the body's target onto the
head is the cheap resolution: the body gives up its own target and follows.

## `update`

**Contract** — the scheduled half. Runs the turning-in-place rule and then lets the
generic selector execute the chosen order. Returns immediately when aiming is disabled —
so a disabled sight manager also stops *executing orders*, not merely stops rotating
bones.

```text
FUNCTION update()
  IF NOT enabled                             RETURN
  IF the creature is moving
    turning_in_place = false
    body.target.yaw  = whatever the order wants        # movement owns the body
    run the selector
    RETURN
  IF NOT turning_in_place
    twist = |body.current.yaw - head.current.yaw|
    limit = max_left_angle IF the head is to the left ELSE max_right_angle
    IF twist > limit
      turning_in_place = true
      body.target.yaw  = head.current.yaw              # start catching up
    ELSE
      body.target.yaw  = body.current.yaw              # hold the feet still
    run the selector
    RETURN
  # already turning in place
  IF |body.current.yaw - head.target.yaw| > epsilon
    body.target.yaw = head.target.yaw                  # chase where the head is *going*
  ELSE
    turning_in_place = false
    body.target.yaw  = body.current.yaw
  run the selector
```

**Invariants** —

- Turning in place is only possible while **stationary**. A walking creature's body yaw is
  owned by the movement layer, and the twist limit is simply not enforced; the animation
  blends absorb it.
- Entry chases the head's *current* yaw; continuation chases the head's *target* yaw. The
  difference is deliberate: entering on the current yaw stops the feet from overshooting a
  head that is still turning, while continuing on the target yaw makes the feet arrive
  together with the head instead of a beat late.
- The asymmetric limits mean a creature turns its feet sooner when looking over one
  shoulder than the other.
- The state is one bit with hysteresis: it starts when the twist exceeds the limit and
  ends only when the body has arrived, not when the twist falls back under the limit.
  Without that the feet would stutter on and off around the threshold.

## `setup`

**Contract** — issues a look order, replacing whatever is running. If the same order is
already the only one running, does nothing — this is the whole reason look orders have an
equality rule ([`sight_action_inline.h`](sight_action_inline.h.md)). Otherwise clears the
set and installs the new order with weight one and an effectively infinite inertia.

```text
FUNCTION setup(order)
  IF more than one order is held THEN clear        # defensive: sight holds at most one
  IF exactly one is held AND it equals `order`  THEN RETURN
  clear
  add order with weight 1 and inertia = forever
```

**Invariants** — the early return is load-bearing at the level of visible behaviour. AI
code re-issues the same look order every planner tick; without the equality check every
tick would restart the order, resetting the glance and aimed-fire state machines and the
creature's head would never settle on anything.

## `remove_links`

**Contract** — forwards the destroyed-entity notice to every held order so none keeps a
reference to it. Cheap and called on every entity destruction, so it walks the (at most
one) held order rather than searching.

## `object_position`

**Contract** — where in the world the currently tracked entity should be aimed at. Applies
the same living-creature rule as the object look order (horizontal from the ground
position, vertical from the centre), then gives the aim-point refinement in
[`sight_manager_target.cpp`](sight_manager_target.cpp.md) a chance to substitute a bone
position. Falls back to the unrefined point if refinement declines.

## `aiming_position`

**Contract** — where the creature is aiming, as a world point, for *any* sight type. Used
by the bone solvers below and by the weapon. Never fails; every sight type has an answer.

```text
FUNCTION aiming_position() -> vector
  fake_distance = 10000                       # direction orders have no point, only a ray
  MATCH order.sight_type
    current_direction    -> position + direction(head.current) * fake_distance
    path_direction       -> position + direction(head.target)  * fake_distance
    direction            -> position + order.payload_vector    * fake_distance
    position, fire_position -> order.payload_vector            # already a point
    object               -> object_position()
    fire_object          -> order.payload_vector               # the predicted aim point
    cover, search, look_over, cover_look_over
                         -> position + direction(head.current) * fake_distance
    animation_direction  -> position + direction(body.current) * fake_distance
```

**Invariants** —

- Direction-only orders are turned into points by projecting ten kilometres. That number
  is not a range, it is "far enough that the bone solver's angle is indistinguishable from
  the pure direction" — and the result is asserted to stay under a hundred kilometres,
  which is the sanity bound on a level's coordinate space.
- The orders that use the head's **current** angle rather than its target are exactly the
  ones whose target is recomputed by a search each tick: aiming at where the head is
  going would chase a target that is itself moving. Path direction is the exception
  because its target is a stable path heading.
- Aimed fire returns the order's payload in both of its internal states. They are written
  as separate arms in the original and compute the same thing; what is load-bearing is
  that the *settled* state returns the frozen point, which is what makes the muzzle hold
  still.

## `process_action` — the no-aimer path

**Contract** — when no bone solver is armed, the additive bone rotations are derived
directly from the head-minus-body angle difference, distributed across the three bones by
the current blend factors, which are themselves crossfaded toward the set the order's
torso flag selects.

```text
FUNCTION process_action(time_delta)
  factors = danger_factors IF order.use_torso_look ELSE free_factors
  FOR each of head, shoulder, spine
    current.factor = step current.factor toward factors[bone] by factor_lerp_speed*time_delta
  angles = ( -(head.pitch - body.pitch), -(head.yaw - body.yaw), head.roll - body.roll )
           each normalized to signed half-turn
  FOR each of head, shoulder, spine
    current.rotation = rotation from (angles scaled by current.factor)
```

**Invariants** — the step toward the target factor is a *clamped linear step*, not an
exponential approach: it moves a fixed amount per second and stops exactly on the target.
That matters because an exponential approach never arrives, and the danger set's zero head
factor must actually reach zero or the neck keeps a residual twist while aiming.

## `compute_aiming`

**Contract** — chooses between three ways of producing the target bone rotations and then
either snaps to them or interpolates toward them. Called once per frame from `Exec_Look`
while aiming is enabled. Calls into the animation layer and may touch the script-visible
animation callbacks, so it is not re-entrant.

```text
FUNCTION compute_aiming(time_delta, angular_speed)
  MATCH aiming_type
    none   -> process_action(time_delta); RETURN
    weapon -> IF order is animation_direction OR the creature has no best weapon
                target rotations = identity
              ELSE
                solve with the weapon aimer against the armed clip, matching
                  spine, shoulder and the weapon's two reference bones,
                  toward aiming_position()
                target.spine    = solved bone 0
                target.shoulder = solved bone 1
                target.head     = identity
    head   -> IF order is animation_direction
                target rotations = identity
              ELSE
                solve with the three-bone aimer (spine, shoulder, head)
                  toward aiming_position()
                target.{spine,shoulder,head} = solved bones 0,1,2
  IF the creature has blend callbacks assigned in either direction
    current rotations = target rotations            # snap
  ELSE IF time_delta is non-zero
    interpolate current toward target at angular_speed
```

**Invariants** —

- **Arming the solver requires a clip.** Both solver arms assert a clip name and a frame
  selector are set; an armed solver with no clip is a programming error, because the
  solver works by *comparing* the desired direction against the pose the clip produces.
- **Bone names come from configuration**, per creature section: the spine bone, the
  shoulder bone, the head bone, and for the weapon solver two bones on the weapon itself.
  They are not hard-coded, because different creature models rig differently.
- **The blend-callback dance is not incidental.** Running an aimer perturbs the model's
  bone callbacks; they are read, removed for the duration of the solve, and restored to
  whichever of the three states they were in (forward blend callbacks, backward blend
  callbacks, or plain bone callbacks). A rebuild that solves without unhooking will fire
  the game-facing animation callbacks — footsteps, casing ejects, hits — an extra time per
  frame.
- **Snap versus interpolate is decided by the callbacks, not by distance.** When blend
  callbacks are active the model is mid-transition and interpolating on top of it fights
  the blend; when they are not, the rotations are eased.
- **An animation-driven look order always yields identity rotations.** The clip owns the
  whole pose and any additive correction would corrupt it.
- The head solver's interpolation uses a *tenth* of the commanded angular speed while the
  animation controller is blending, which slows the head's correction through a
  transition so it does not visibly whip.

**Notes** — the head-solver path logs a diagnostic when it finds no animation movement
controller, and the original marks it as reporting a real bug. The behaviour when it
happens is to skip the interpolation entirely and leave the current rotations alone, which
is the correct degradation: a stale correction is better than a wrong one.

## `slerp_rotations`

**Contract** — moves each of the three current bone rotations toward its target along the
shortest arc, at a bounded angular rate, in a single time step. Snaps exactly onto the
target when the step would reach or overshoot it, so the rotation actually terminates.

```text
FUNCTION slerp_one(time_delta, angular_speed, current, target)
  difference = target composed with the inverse of current
  (axis, angle) = axis-angle of difference
  IF angle is zero
    current = target; RETURN
  test_angle = angle clamped to (epsilon, half turn)
  speed = angular_speed
  IF angular_speed > one-tenth turn per second AND test_angle < that
    speed = test_angle                          # short hops take about a second
    IF speed < one-360th turn per second THEN speed = that floor
  factor = clamp(time_delta / (angle / speed), 0, 1)
  IF factor is ~1
    current = target; RETURN
  current = spherical interpolation from current to target by factor
```

**Invariants** —

- The interpolation parameter is `time_delta / time_to_cover_the_angle`, which makes the
  *angular rate* constant rather than the parameter rate. A plain fixed-factor blend would
  make large corrections start fast and crawl at the end.
- The speed rewrite for small angles is the interesting decision: when the commanded speed
  is fast but the remaining angle is small, the speed is replaced by the *angle itself*,
  numerically equal to covering it in one second. Small corrections therefore take a
  roughly constant time regardless of how small they are, instead of completing instantly
  and popping. The floor stops that from degenerating to no motion at all.
- The exact-snap on a near-unit factor is not an optimization; without it the rotation
  approaches the target asymptotically and the aimer never reports settled.

## `adjust_orientation`

**Contract** — resets both current and target bone rotations to identity and re-derives
the creature's body and head angles *from its world transform*, with roll forced to zero
and head equal to body. Used when aiming is re-enabled after an animation has moved the
creature.

**Invariants** — roll is discarded because the angle pairs this system carries have no
meaningful roll for a standing creature; leaving an animation's roll in them would make
the very next frame's correction try to un-roll the model. Setting head equal to body is
what stops the head from snapping back to a pre-animation direction.

## `enable`

**Contract** — toggles aiming. Idempotent. Turning it *off* does nothing beyond setting the
flag, so the bones keep whatever correction they had. Turning it *on* re-synchronizes the
angles with the model's actual orientation — but only if an animation movement controller
exists, because otherwise the transform was never taken away from this system and is
already consistent.

**Invariants** — the asymmetry is the point. Disabling is instant and lossy; enabling must
pay the cost of re-reading the world transform, or the creature snaps back to where it was
looking before the animation.

## `reinit` / `reload` / `Load`

**Contract** — `reinit` re-enables aiming, clears the turning-in-place state and seeds the
three blend factors from the *free* set (a creature comes back from a reset standing
relaxed, not aiming). `reload` reads the two torso twist limits from the creature's
configuration section, defaulting to a quarter turn left and a sixth of a turn right when
absent. `Load` does nothing — the surface exists because the base class demands it.
