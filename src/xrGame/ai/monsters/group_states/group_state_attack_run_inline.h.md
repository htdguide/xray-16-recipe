# src/xrGame/ai/monsters/group_states/group_state_attack_run_inline.h

> The charge: run at where the enemy *will be*, offset by a wandering vector so the pack does not
> converge on one line, and approach from the direction the squad assigned.

**Needs** — [`group_state_attack_run.h`](group_state_attack_run.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../state.h`](../state.h.md) · [Level graph — chapter 14](../../../../xrAICore/README.md)
**Used by** — [`group_state_attack_run.h`](group_state_attack_run.h.md)
**Tier floor** — T3: vector arithmetic and a navigation-mesh reachability probe each tick

## Purpose

Where the pack attack brain decides *whether* to charge, this state decides *where to run*. Three
independent mechanisms are layered onto "run at the enemy", and each solves a specific visible
problem:

1. **Prediction** — running at the enemy's current position means always arriving behind a moving
   target. The state estimates the enemy's velocity and leads it.
2. **Interception wander** — a pack that all leads perfectly converges into a single file. Each
   member carries a slowly rotating random offset, so the lines fan out.
3. **Encirclement** — for a window at the start of a charge, the creature is required to *arrive
   facing a particular direction*, taken from the squad's command. That is what makes a pack
   surround rather than queue.

## State

```text
RECORD GroupAttackRunState
  # interception: a unit vector in the horizontal plane, redrawn on a timer
  intercept        : vector
  intercept_tick   : int
  intercept_length : int      # this offset's lifetime: 2000..6000 ms (3000..7000 on entry)

  # prediction: a coarse finite difference of the enemy's position
  memorized_tick   : int
  memorized_pos    : vector
  predicted_vel    : vector   # clamped componentwise to 10 units/second

  # encirclement: a direction to arrive from, and how long to insist on it
  encircle_dir     : vector
  encircle_time    : int      # 2000..6000 ms, or 0 when the cooldown has not expired
  next_encircle_tick : int    # earliest time the next charge may encircle

  time_path_rebuild : int     # declared; never read
```

The encirclement cooldown is **not** reset per activation: it lives across charges, so a creature
that has just encircled charges straight the next time. That is what stops the behaviour from
reading as a tic.

## `initialize`

**Contract** — prime the path builder; draw a fresh interception direction with a 3–7 second
lifetime; anchor the prediction at the enemy's current position with zero velocity; and decide
whether this charge encircles.

```text
FUNCTION initialize()
  path.prepare()

  intercept = random horizontal unit vector       # degenerate case falls back to +Z
  intercept_length = 3000 + random(0..3999)

  memorized_pos = enemy.position; predicted_vel = 0

  IF now() > next_encircle_tick
    encircle_time      = 2000 + random(0..3999)
    next_encircle_tick = now() + 8000 + random(0..7999)
  ELSE
    encircle_time = 0                             # this charge goes straight in

  encircle_dir = intercept
  IF in an active squad AND the squad's command for me is "attack"
    encircle_dir = the squad's assigned direction
  normalize; degenerate case falls back to +Z
```

**Notes** — the encirclement direction defaults to the creature's own random interception vector
and is *overridden* by the squad when there is one. So a lone creature still arcs, and a pack arcs
in assigned directions. That fallback is what lets this state be used by solitary creatures
unchanged.

Every one of the six random draws has a degenerate case handled explicitly — a zero-length vector
becomes a fixed axis. That is not defensive noise: a zero direction fed to the path builder
produces an unconstrained arrival orientation, which reads as the creature snapping round on
arrival.

## `execute`

**Contract** — recompute the target point and hand it to the path builder, along with the run
action, the aggressive acceleration profile, the distance-scaled rebuild interval and the
aggression sound. Each tick, refresh the interception offset if its lifetime expired and the
velocity estimate if a quarter second has passed. Allocates nothing.

```text
FUNCTION execute()
  IF now() > intercept_tick + intercept_length
    redraw intercept; intercept_length = 2000 + random(0..3999)

  IF now() > memorized_tick + 250
    predicted_vel = (enemy.position - memorized_pos) scaled to per-second
    clamp each component to 10                    # an enemy cannot be led faster than this
    re-anchor the memory

  self_speed        = my run velocity from the section
  distance_to_enemy = |enemy.position - self.position|

  prediction_time = distance_to_enemy / self_speed, capped   # how long until I arrive
  linear_prediction = enemy.position + predicted_vel * prediction_time
  turning_left = the linear prediction lies to the left of the straight line to the enemy

  # instead of aiming at the linear prediction, rotate the straight line toward it
  angle = 0.5 * prediction_time * |predicted_vel| / distance_to_enemy
  heading = heading_to_enemy rotated by +/- angle
  radial_prediction = self.position + direction(heading) * distance_to_enemy

  target = radial_prediction + intercept * distance_to_enemy * 0.5

  vertex = navigation.first_reachable_vertex_from(self.vertex, toward target)
  IF no valid vertex
    target = enemy.position; vertex = enemy.vertex     # give up leading; run straight at it

  request_action(run)
  animation.accelerate(aggressive, braking = false)
  path.target = (target, vertex)
  path.rebuild_every = my distance-scaled attack rebuild interval
  path.use_covers = false
  path.extrapolate = true
  path.try_min_time = (distance_to_enemy >= 5)

  IF the enemy is outside the home region
     OR we are still inside this charge's encirclement window
    path.arrive_facing(encircle_dir)
  ELSE
    path.arrive_facing(none)
```

**Notes** — the *radial* rather than linear prediction is the load-bearing trick. Aiming straight
at the linear prediction makes a creature cut a chord and then turn hard; rotating the heading
toward it keeps the creature the same distance from its target and sweeps the approach, which is
both smoother and — because the rotation is proportional to how fast the enemy is moving relative
to the gap — self-correcting. The factor of one half on the rotation angle is authored in code and
nothing records how it was chosen.

The interception offset is added *scaled by the current distance*: far away it is a large
sideways bias, near the enemy it shrinks to nothing. So the fan-out closes into a convergence, and
the pack does not miss.

The velocity clamp at 10 units per second is a defence against the enemy teleporting — a level
change or a script move produces an enormous finite difference, and without the clamp the creature
would sprint to the far side of the map for a quarter second.

Preferring minimum *time* over minimum distance only beyond 5 units is the one path-builder
parameter that changes mid-charge; inside that radius the creature takes the direct route, because
a time-optimal detour at melee range looks like indecision.

Requiring an arrival direction while the enemy is *outside* the home region, and dropping it once
the enemy is inside, is subtle and deliberate: a pack surrounds an intruder who is approaching its
territory, and simply piles on once the intruder is inside.

## `finalize` / `critical_finalize`

**Contract** — both stop the path extrapolation this state turned on. The extrapolation setting
outlives the state otherwise, and would make the next state's paths overshoot.

## `check_completion` / `check_start_conditions`

**Contract** — two thresholds from the creature's melee tracker: the charge finishes inside the
tracker's minimum distance, and may start outside its maximum distance. Both distances come from
the creature's section via the tracker, so the band is authored per creature.

**Notes** — the band between the two is the reason the charge and the melee state do not
oscillate: there is a distance range in which neither predicate fires and whichever state is
active stays active.

The file keeps a full earlier version of the execute body, commented out, which aimed directly at
the enemy's remembered position with cover-preferring paths. The present version replaced it; the
comparison is the clearest statement of what the prediction and interception machinery bought.
