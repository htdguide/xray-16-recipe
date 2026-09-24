# src/xrGame/ai/monsters/states/monster_state_attack_on_run_inline.h

> The circling attack: aim at an intercept point offset to one side of where the enemy will be, strike in passing when the geometry lines up, then swing wide and come back from the other side. It never terminates; the brain above it decides when the fight is over.

**Needs** — [`monster_state_attack_on_run.h`](monster_state_attack_on_run.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_attack_on_run.h`](monster_state_attack_on_run.h.md)
**Tier floor** — T2: estimates the target's velocity and solves a right-triangle intercept each tick against the navigation mesh

## Purpose

The most elaborate behaviour in chapter 24 and the newest. A creature with this state does not
approach and bite; it *orbits*, and the fight becomes a series of passes. Six mechanisms make it
work, and each is a separable idea:

1. **Enemy prediction** — a velocity estimate with a one-second sampling interval, used to lead
   the target.
2. **A two-phase orbit** — close in, then deliberately swing wide, alternating.
3. **A tangent solve** — the destination is not the enemy but a point offset by an authored
   attack radius, computed so the creature passes at exactly that distance.
4. **An alternating pass side** that is forced to flip on a timer, so the creature does not
   circle one way forever.
5. **A strike timed from the animation**, not from the state, using how long into the clip the
   hit lands to decide how early to commit.
6. **A validity check and a fallback scan**, because every computed destination must be a place
   the creature can actually stand.

Nearly every number is authored on the creature, which is what lets one behaviour serve several
very different creatures.

## `execute` — the tick

**Contract** — one tick. Runs the five update steps in a fixed order, then issues the movement
request. Never completes.

```text
FUNCTION execute()
  calculate_predicted_enemy_pos()
  update_movement_target()
  update_try_min_time()          # maintains fields nothing reads; see Notes
  update_attack()
  update_aim_side()

  set_action(run)
  animation.accelerate(aggressive, braking: false)
  path.target           = (target, target_vertex)
  path.rebuild_interval = IF attacking THEN 20 ELSE 150 milliseconds
  path.use_covers       = true
  path.cover_weights    = (0.1, 30, 1, 30)
  path.try_min_time     = (phaze == go_close)
  state_sound           = aggressive
  path.extrapolate      = true
  path.use_destination_facing = false
```

**Invariants**

- **The ordering matters.** Prediction feeds the destination solve; the destination solve may
  change the phase, which the strike test then reads; the side choice runs last so that the
  strike can force a recalculation before committing to an animation.
- **The replan rate is seven times faster while striking** (twenty milliseconds against a
  hundred and fifty). During a pass the creature must track its target closely; between passes a
  coarse route is enough and the cost is saved. This is the one place in the chapter where the
  replan rate is behaviour-dependent rather than creature-dependent.
- **Fast routing is requested only while closing.** During the disengage the creature takes the
  geometrically shorter arc instead, which is what makes the swing-wide read as deliberate
  rather than as fleeing.
- **The squad-facing block from the plain approach is commented out here.** A circling creature
  does not take arrival directions from its squad, so a pack using this state does not
  coordinate its approach angles — each member circles independently.

## `calculate_predicted_enemy_pos` — leading the target

**Contract** — estimates where the enemy will be when the creature reaches it. Cheap when far,
a velocity extrapolation when near. Never returns a point coincident with the creature.

```text
FUNCTION calculate_predicted_enemy_pos()
  IF distance(enemy, me) > far_radius * 2
    predicted = enemy.position               # too far for prediction to mean anything
    RETURN

  time_to_reach = distance(enemy, me) / my current speed     # 0 if I am stationary

  IF more than 1 second since the last velocity sample
    IF this is not the first sample
      IF less than 2 seconds since the last one
        velocity = (enemy.position - last_sampled_position) / elapsed
      ELSE
        velocity = zero                      # the gap was too long to trust
    record this sample

  predicted = enemy.position + velocity * time_to_reach * creature.prediction_factor

  IF predicted is within 1 centimetre of me
    predicted = enemy.position
    IF still within 1 centimetre
      nudge predicted one unit along a fixed axis
```

**Invariants**

- **The velocity is sampled at one-second intervals, not per tick.** A per-tick difference is
  dominated by the enemy's animation jitter and by scheduling noise; a one-second baseline gives
  a stable heading. The cost is that the estimate lags a direction change by up to a second,
  which is exactly the exploitable window the mechanic wants — a player who changes direction
  makes the creature miss.
- **A sample gap over two seconds discards the estimate** rather than dividing by it. The
  creature is scheduled at a rate that degrades with distance, so a long gap means the estimate
  would be built from a stale position.
- **The prediction strength is authored per creature.** A creature with a factor of zero attacks
  where the enemy is; a high factor over-leads and can be sidestepped. It is the single number
  that tunes how hard this creature is to dodge.
- **The degenerate-coincidence guard is not cosmetic.** Every downstream computation normalises
  the vector from the creature to the predicted point, so a zero-length vector would produce a
  meaningless direction. The final fallback — nudging along an arbitrary axis — is ugly but
  bounded, and a rebuild should instead fall back to the creature's own facing.

## `update_movement_target` — the phase machine and the tangent solve

**Contract** — advances the orbit phase, computes the destination, validates it against the
navigation mesh, and falls back when it is unreachable. The longest routine in the chapter.

### The phase transitions

```text
IF phaze == go_prepare        # disengaging
  return to go_close when ANY of:
    - the authored preparation time has elapsed
    - the predicted enemy is within 3 units and I have been preparing 3 seconds
    - I have travelled more than twice the far radius from where I began preparing
    - the enemy is now further than the far radius plus 3 units
ELSE IF phaze == go_close     # closing
  switch to go_prepare when EITHER of:
    - the enemy is more than 140 degrees behind me, the predicted point is within 4 units,
      and I have been closing at least 3 seconds        # I have overshot; swing wide
    - I have been closing longer than the authored maximum
```

**Invariants** — the overshoot test is the heart of the orbit. A creature that has run past its
target has the target *behind* it and *close*, and rather than turning in place it commits to a
wide arc and comes back — which is what makes the behaviour read as circling rather than as
lunging repeatedly. The three-second floors on both transitions prevent a creature from
oscillating between phases when the geometry is marginal.

### The destination

Four cases, in the order they are tried:

**Finishing the current pass.** After a strike is launched, a flag holds the destination fixed
until the creature has either reached it or spent a second trying, and then forces a disengage.
So a strike always carries through instead of being re-aimed mid-swing.

**Disengaging.** The destination is one far-radius from the predicted enemy position, on a
bearing rotated away from the creature by at least thirty degrees (or by whatever angle a
five-unit offset subtends at the far radius, whichever is larger), toward the side chosen when
the disengage began. That rotation is what turns a straight retreat into an arc.

**Closing from outside the attack radius — the tangent solve.** The creature wants to pass the
enemy at exactly the authored attack radius rather than run into it. Treating (self, enemy,
pass-point) as a right triangle with the attack radius as the opposite side, it solves for the
bearing to the tangent point, then aims three units *beyond* it so the run carries through
rather than stopping on the tangent.

```text
adjacent = sqrt(range^2 - attack_radius^2)
cos a    = adjacent / range
sin a    = (attack_radius / range) * (attack side == right ? -1 : +1)
direction = the bearing to the predicted enemy, rotated by a
target    = me + normalize(direction) * (adjacent + 3)
```

**Closing from inside the attack radius.** No tangent exists, so the creature instead runs
*perpendicular* to the line to its enemy — the side chosen to be whichever the creature is
already facing — for the distance that carries it back out to the far radius. This is the
"too close, break away sideways" case.

### Validation and fallback

The computed destination must be a place the creature can stand: it must resolve to a
navigation-mesh cell whose surface is within four units of the point's height. While closing,
the *route from the destination back to the enemy* must additionally be clear, which is what
stops the creature choosing a pass-point on the far side of a wall.

On failure: while closing, the creature abandons the offset entirely and heads straight for the
enemy's own cell — and if it is already standing there, disengages. While disengaging, it scans
eight bearings at forty-five-degree intervals around the enemy at the far radius and takes the
first that is valid, falling back to the enemy's own cell if none is.

**Invariants** — **the fallbacks are what keep the behaviour from breaking indoors.** The whole
orbit assumes open ground; in a corridor almost every offset point is invalid, and the fallback
collapses the creature into a plain charge. That degradation is deliberate and a rebuild that
fails the tick instead gets a creature that stands still in every room.

## `update_aim_side` — forcing the creature to alternate

**Contract** — chooses which side the next pass goes by. Computes which side the enemy is
currently on, and then, **once per authored interval**, either adopts that side or its opposite.

```text
FUNCTION update_aim_side()
  natural = the side my enemy is currently on
  IF now > attack_side_chosen_time + creature.update_side_period
    side = IF side == natural THEN the opposite ELSE natural
    attack_side_chosen_time = now
```

**Invariants** — the rule reads strangely and is deliberate: **if the creature is already
passing on the side the enemy happens to be on, it flips to the other.** The effect is that the
creature cannot keep circling one way — each interval it is forced to cross over. A creature
that always took the natural side would orbit in one direction and be trivially tracked.

The interval is authored per creature, in seconds, and is one of the few numbers that visibly
changes how a fight feels.

**Notes** — the routine ends with a conditional return that does nothing, and the entire body
runs identically whether or not a strike is in progress. Vestigial.

## `update_attack` — launching a strike

**Contract** — either ends a strike in progress, or looks for one to start. A strike may only
begin while closing, while not jumping, and while no animation override is already in force.

```text
FUNCTION update_attack()
  IF attacking
    IF now > attack_end_time
      choose_next_attack_animation()
      attacking = false
      clear the animation override if it is still one of ours
    RETURN

  IF is_jumping OR an override animation is active OR phaze != go_close
    SKIP to the jump edge below

  FOR EACH enemy I remember
    position = (the primary enemy ? predicted_enemy_pos : its actual position)
    horizontal_range = |position - my position| with height removed

    lead_distance     = animation_hit_time[attack_side] * my current speed
    allowed_window    = (lead_distance + melee_max_distance) * 0.9
    disallowed_window = (lead_distance + melee_min_distance * 0.5) * 0.9

    IF horizontal_range < disallowed_window AND this is the primary enemy
       AND I have been closing 3 seconds
      switch to go_prepare                 # too close to strike: swing wide instead

    IF facing within 30 degrees AND disallowed_window < horizontal_range < allowed_window
      strike this one ; BREAK

  IF a strike was found
    creature.on_attack_on_run_hit()
    attacking = true
    hold the current destination for up to one second, then force a disengage
    update_aim_side()                       # recompute the side before committing the clip
    attack_end_time = now + the chosen clip's length
    override the animation with the chosen side and variant

  IF I was jumping and am no longer
    switch to go_prepare
```

**Invariants**

- **The commit distance is derived from the animation, not authored.** The creature multiplies
  *how long into the clip the hit lands* by *how fast it is currently moving* to get how far it
  will travel before the blow connects, and adds the melee reach. So a fast creature commits
  earlier than a slow one and a creature with a late-landing clip commits earlier than one with
  an early-landing clip — automatically, from data. This is the single most transferable idea on
  the page and a rebuild that hard-codes a commit distance loses it.
- **The window is a band with a floor, not a threshold.** Below the lower bound the creature is
  already too close for the swing to land, and the response is not to strike harder but to
  *disengage* — which is what converts a missed approach into another orbit.
- **The ten-percent shrink** applied to both bounds is a safety margin against the speed
  changing between the decision and the hit. Compiled in, with no recoverable derivation.
- **Any remembered enemy may be struck, not only the current target.** The creature sweeps its
  whole enemy memory and hits whichever one the geometry favours, which is why these creatures
  cut through a crowd. Only the primary enemy is *predicted*, though; the others are taken at
  their current positions, because the prediction machinery tracks one target.
- **Range is measured with height removed.** A target above or below is struck as if level, which
  is what lets these creatures hit a player on a step.
- **Landing from a jump forces a disengage**, detected on the edge rather than on the flag, so a
  creature that leaps into a fight starts its orbit from a clean phase.

## `choose_next_atack_animation`

**Contract** — picks, for each of the two sides, a random variant among the clips authored for
that side, and records how far into each the hit lands. Called at entry and after every
completed strike, so consecutive passes do not repeat the same swing.

**Invariants** — asserting at least one variant exists per side is the state's only hard
requirement on a creature's animation data; everything else degrades.

## `check_control_start_conditions` — the veto

**Contract** — whether a motion-control component may seize the creature during this state.

- **The anti-aim step** is allowed only while closing and not striking — it is a dodge, and a
  creature mid-swing or mid-disengage must not dodge.
- **The rotation jump** is allowed only while closing, within ten units, and only if a coin
  flipped at the start of the current disengage came up right. So the leap is available at most
  once per orbit and not every orbit, which is what keeps it surprising.
- Everything else is permitted.

**Notes** — the ten-unit range is compiled in. The coin is re-flipped in `set_movement_phaze`
each time a disengage begins.

## `set_movement_phaze`

**Contract** — records the new phase and its timestamp. Entering the closing phase clears the
side-choice timer, forcing the side to be recomputed on the next tick. Entering the disengage
records the starting point, re-flips the rotation-jump coin, and picks which side to swing
toward — **from the cross product of the direction to the enemy and the creature's own facing**,
so the creature arcs the way it is already turning rather than reversing.

**Notes** — a commented-out line beside it would have chosen the side at random. The geometric
choice is what makes the arc continuous.

## `initialize`, `finalize`, `critical_finalize`, `check_start_conditions`, `check_completion`, `remove_links`

**Contract** — entry clears the hit stamp, prepares the path builder, invalidates the
destination, picks a random starting side and a random rotation-jump coin, zeroes the prediction
history, enters the closing phase and chooses the first animations. Both exits clear the
animation override — **both**, because a creature displaced mid-swing would otherwise keep the
override forever.

`check_start_conditions` is identical to the charge's gate: within the creature's authored
start distance, outside the melee minimum, facing within thirty degrees.

`check_completion` **always returns false.** The state never ends of its own accord; only the
brain above it can displace it. The test that would have ended it — no longer moving, or a hit
landed — is present and commented out. That is the deliberate difference from the charge: a
charge is one pass, this is a fight.

`remove_links` must additionally clear the secondary attack target, since it is the one entity
reference this state caches outside the enemy manager.

## Notes

Three things in this file are inert and are named here so a reader does not hunt for their
effect:

- `update_try_min_time` re-rolls a random boolean on a random three-to-six-second period, and
  **nothing reads the result**. The movement request derives its fast-routing flag from the
  phase instead. The intended behaviour — randomly alternating between fast and short routing —
  does not happen.
- A namespace constant naming a maximum duration for the unused third phase, and another naming
  a side-update period, are both shadowed or unreferenced; the live values come from the
  creature.
- `can_do_preparation` and `last_update_time` are declared and never used.

The debug instrumentation is compiled into developer builds only and publishes the phase, the
attack window tests and the destination-hold flag to the on-screen tree; it is the intended way
to diagnose this state and worth keeping in a rebuild.
