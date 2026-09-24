# src/xrGame/ai/monsters/chimera/chimera_attack_state_inline.h

> The chimera fight: circle behind the enemy, pounce, and after a run of pounces spend a couple of them just repositioning — with a crouched pause whenever it gets behind the enemy unseen.

**Needs** — [`chimera_attack_state.h`](chimera_attack_state.h.md) · [`chimera.h`](chimera.h.md) · [`state.h`](../state.h.md) · [`control_jump.h`](../control_jump.h.md) · [`control_direction_base.h`](../control_direction_base.h.md) · [`control_path_builder_base.h`](../control_path_builder_base.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`AISpaceBase.hpp`](../../../../xrAICore/AISpaceBase.hpp.md)
**Used by** — [`chimera_attack_state.h`](chimera_attack_state.h.md)
**Tier floor** — T2: per-tick geometry against the navigation mesh and the jump ability's reachability test

## Purpose

Every other creature in the chapter attacks by closing distance and running a melee clip. The chimera never does: it moves only to get into pounce range, and every blow it lands is a pounce. That makes this state a target-selection problem rather than a chase, and almost all of it is about *where to jump to*, not *when to hit*.

The fight has a rhythm the player can read: a run of damaging pounces at the enemy, then a couple of pounces that deliberately land somewhere else, then the run again. That is what stops a chimera from being an unbroken blender, and it is authored — the two counts come from the creature's configuration section.

## State

See [`chimera_attack_state.h`](chimera_attack_state.h.md) for the field list. The load-bearing invariants:

- `phase` is `free` unless the creature is committed to a specific pounce. While `rotating` or `winding_up`, the state has taken *path* and *movement* control away from the ordinary path follower and must give both back on every exit, or the creature is left frozen.
- `allow_jump` is a latch, not a state: it is raised immediately before asking the jump ability anything and lowered immediately after. See the arbitration hook.
- `attack_jumps_done` reaching its authored maximum is what puts the creature into repositioning mode; `prepare_jumps_done` reaching its maximum resets *both* counters at once.

Authored elsewhere, read here every tick: `attack_radius`, `prepare_jump_timeout`, `attack_jump_timeout`, `stealth_timeout`, `num_attack_jumps`, `num_prepare_jumps` — all from the creature's section, all with code defaults, see [`chimera.cpp`](chimera.cpp.md).

Fixed in code:

```text
scan_points        = 10          # candidate directions around the enemy for a pounce
scan_angle         = 36 degrees  # one full turn divided by scan_points
move_scan_points   = 8           # candidate directions when merely repositioning
behind_cone        = 30 degrees  # how tightly "behind the enemy" is defined
stealth_min_range  = 6 world units
aim_tolerance      = 20 degrees  # when a rotation counts as finished
side_hold          = 4000 ms     # how long a chosen circling side is kept
close_enough       = 5 world units  # skip circling, head straight for the spot behind
min_scan_separation= 7 world units  # a pounce target must be at least this far away
jump_nudge         = 1 world unit   # see `correct_jump_pos`
```

## `initialize`

**Contract** — Entering the fight. Arms the melee bookkeeping, derives `min_run_distance` from the creature's attack radius and the jump ability's maximum reach, takes a handle to the jump ability, and zeroes every counter and phase. Asserts the jump ability exists — a chimera without one cannot fight at all.

## `min_run_distance` — the one derived number

**Contract** — The distance beyond which the creature stops trying to pounce and simply runs at the enemy. Computed once per entry from two authored quantities.

```text
FUNCTION calculate_min_run_distance() -> real
  h     = attack_radius * sin(scan_angle)
  leg_1 = attack_radius * cos(scan_angle)
  reach = jump_ability.max_distance
  ASSERT h < reach                # otherwise no scan point is ever reachable
  leg_2 = sqrt(reach * reach - h * h)
  RETURN leg_1 + leg_2
```

**Notes** — The geometry: the creature wants to land on a ring of radius `attack_radius` around the enemy, at one of the scan directions. `h` is how far the *first* scan direction sits off the straight line to the enemy; `leg_1 + leg_2` is then the farthest the creature can stand and still have that point inside its jump reach. Past that, no candidate is reachable and scanning would be wasted work, so it runs instead. The assertion is a real data constraint: a section whose attack radius is large relative to the model's jump reach makes a chimera that can never open a fight.

Two of the three trigonometric terms are named for a *half* scan angle in the source and computed from the whole one. Whether the intent was half the angle — which would make the ring reachable from farther out — is not recoverable; the shipped behaviour is the whole angle.

## `execute` — the tick

**Contract** — One decision and one movement command per tick. Never blocks. Ends by handing the path follower a target; the actual pounce is issued by the jump ability, which takes the creature over for the duration of the flight.

```text
FUNCTION execute()
  enemy_pos, self_pos, self_to_enemy, distance := measure()
  behind_enemy      = angle_between(self_to_enemy, enemy_facing) < behind_cone
  can_sneak         = behind_enemy AND distance >= stealth_min_range
  repositioning     = (attack_jumps_done == num_attack_jumps)

  may_prepare_jump  = repositioning     AND now() > last_jump_tick + prepare_jump_timeout
  may_attack_jump   = NOT repositioning AND now() > last_jump_tick + attack_jump_timeout

  IF phase == rotating            do_rotating()
  ELSE IF phase == winding_up     do_winding_up()
  ELSE IF airborne()              # nothing: the jump ability owns the creature
  ELSE IF distance >= min_run_distance
       target = enemy's navigation vertex and its position
  ELSE IF may_prepare_jump OR may_attack_jump
       IF select_target_for_jump(may_attack_jump ? attack : reposition)
          jump_target = target ; phase = rotating
          IF can_sneak  stealth_until = now() + stealth_timeout
          take path control and movement control from the path follower
          stop both
          play the fast turn clip toward the target
       ELSE
          reposition_only = true
  ELSE reposition_only = true

  IF reposition_only
     IF select_target_for_move()  target_vertex = vertex_of(target)
     ELSE                         target = enemy's vertex position

  action = run
  path.use_destination_orientation = false
  path.try_min_time = false
  acceleration = aggressive, braking off
  path.rebuild_interval = 250 ms
  path.extrapolate = true
  path.target = (target, target_vertex)
```

**Notes** — The final block runs on *every* path through the tick, including the ones that committed to a pounce. That is deliberate: while rotating and winding up, the path follower has been taken over and ignores the target, but the moment control is released the target is already current and the creature does not stutter.

## `do_rotating` — swinging round to face the pounce

**Contract** — Holds the creature still and turns it toward the chosen target. When aimed, re-validates the target and either commits to the wind-up or abandons the pounce.

```text
FUNCTION do_rotating()
  target = jump_target
  IF is_attack_jump
     target = correct_jump_pos(enemy_pos)          # nudge past the enemy
     IF target is not on the navigation mesh
        target = position of the enemy's own vertex
     target_vertex = vertex_of(target)

  aimed     = |heading_to(target) - own_heading| < aim_tolerance
  sneaking  = now() < stealth_until AND NOT enemy_can_see_me_now()
  IF NOT sneaking  stealth_until = 0

  clear any override animation
  IF NOT aimed OR sneaking
     IF aimed AND sneaking   play the crouched prepare clip
     ELSE                    play the fast turn clip toward target
     turn toward target
  ELSE
     release path control and movement control
     ok = jump_is_possible(target)
     IF NOT ok AND is_attack_jump
        target = jump_target ; target_vertex = vertex_of(target)
        ok = jump_is_possible(target)
     IF ok
        phase = winding_up
        phase_end_tick = now() + length_of(launch_clip)
     ELSE
        phase = free
     circle_side = undecided
     stealth_until = 0
```

**Notes** — The stealth branch is the chimera's most recognisable behaviour. Having got behind the enemy at range and aimed, it does *not* pounce immediately: it holds the crouched clip until either the stealth window expires or the enemy turns and sees it. The window is re-armed only when the pounce was chosen from behind, so the creature does not crouch when it is being watched.

An attack pounce re-aims at the enemy's *current* position on every tick of the rotation, because the enemy is moving; a repositioning pounce keeps the fixed spot it chose. The fallback to the enemy's own navigation vertex matters when the enemy is standing somewhere the mesh does not cover exactly — the creature would rather land on the nearest walkable cell than abandon the pounce.

## `do_winding_up` — the launch

**Contract** — Plays the launch clip for its own length, then issues the pounce.

```text
FUNCTION do_winding_up()
  IF now() <= phase_end_tick
     play the launch clip
     RETURN
  clear the override animation
  IF issue_jump(target, is_attack_jump)
     last_jump_tick = now()
     IF is_attack_jump
        attack_jumps_done = attack_jumps_done + 1
     ELSE IF repositioning
        prepare_jumps_done = prepare_jumps_done + 1
        IF prepare_jumps_done == num_prepare_jumps
           attack_jumps_done = 0 ; prepare_jumps_done = 0
  target = position of the enemy's vertex
  phase  = free
```

**Notes** — A pounce that fails to launch still ends the phase and still costs the timeout, so a chimera blocked by geometry backs off and tries a different angle rather than jamming. Note that the counters only advance on a *successful* launch, so a run of failures never pushes the creature into repositioning mode.

## `issue_jump`

**Contract** — Asks the jump ability to launch. Refuses if the creature's heading is more than about a radian off the target's direction — the rotation phase is supposed to have fixed that, and launching anyway would slew the creature in mid-air. Raises and lowers the arbitration latch around the call. Returns whether a pounce started.

## `select_target_for_jump`

**Contract** — Chooses where the next pounce lands. Two modes: an attack pounce aims at the enemy, a repositioning pounce aims at a spot on the ring around the enemy, preferring the spot directly behind them.

```text
FUNCTION select_target_for_jump(mode) -> bool
  IF mode == attack AND select_target_for_attack_jump()   RETURN true

  is_attack_jump = false
  ASSERT distance_to_enemy < min_run_distance
  ASSERT attack_radius >= 4
  radius       = 3 + random integer in [0, attack_radius - 3)
  behind_point = enemy_pos - enemy_facing * radius

  IF jump_is_possible(behind_point)
     target = behind_point ; RETURN true

  FOR index IN 0 .. scan_points-1, stepping by two
     # alternate sides, with the leading side re-drawn each pair
     side  = coin flip, alternating within the pair
     angle = scan_angle * (index/2 + 1) * (side == left ? -1 : +1)
     point = enemy_pos + rotate(behind_point - enemy_pos, angle)
     IF distance(self_pos, point) >= min_scan_separation
        AND jump_is_possible(point)
        target = point ; RETURN true

  RETURN select_target_for_attack_jump()      # nothing on the ring: go for the enemy
```

**Notes** — The ring radius is re-drawn at random between three units and the attack radius on every pounce, which is why a chimera's circling never settles into an orbit the player can time. The scan walks outward in pairs, one candidate each side of the behind-point, so nearby angles are tried before distant ones and the creature stays roughly behind the enemy. The minimum separation stops it from pouncing a couple of steps sideways, which reads as a twitch rather than a leap. Falling back to an attack pounce means a cornered chimera commits rather than stalling — the last resort of a repositioning pounce is a real one.

The loop increments its index twice per iteration — once in the step and once in the body — so it examines five pairs rather than ten. The effect is that the outermost scan angles are never tried; whether that was intended is not recoverable.

## `select_target_for_attack_jump`

**Contract** — Tries the nudged enemy position first, the plain enemy position second, and gives up. Sets the attack flag on success.

## `correct_jump_pos`

**Contract** — Pushes a target one unit further along the line from the creature to it.

**Notes** — Aiming exactly at the enemy's feet lands the creature short, because the pounce arc is measured to the point and the creature has a body. Overshooting by a unit lands it *on* the enemy. The unit is fixed in code and is the difference between a pounce that connects and one that stops a pace away.

## `jump_is_possible`

**Contract** — Three independent tests, all of which must pass: the jump ability accepts the arc, the target is a valid position on the navigation mesh, and the arc does not pass through level geometry (ignoring the enemy's own body, which it is allowed to hit). Raises and lowers the arbitration latch around the first.

## `select_target_for_move`

**Contract** — Chooses where to run when not pouncing: circle the enemy toward the point behind them, from a side that is held for a few seconds so the creature does not oscillate.

```text
FUNCTION select_target_for_move() -> bool
  behind_point = enemy_pos - enemy_facing * attack_radius
  IF distance(self_pos, behind_point) < close_enough
     target = behind_point ; RETURN true

  IF circle_side == undecided OR now() > circle_side_until
     side = whichever side of the line to the enemy the behind-point lies on
     with probability one half, take the other side instead
     circle_side = side
     circle_side_until = now() + side_hold

  offset = -attack_radius * unit(self_to_enemy)     # the ring point nearest us
  FOR index IN 1 .. move_scan_points
     angle = (one full turn / move_scan_points) * index, signed by circle_side
     point = enemy_pos + rotate(offset, angle)
     IF point is on the navigation mesh
        target = point ; RETURN true
  RETURN false
```

**Notes** — The coin flip is what keeps the chimera from always taking the geometrically shorter way round; half the time it deliberately takes the long way, which is why two chimeras fighting the same target do not overlap. The side is then *held*, so the creature commits to an arc instead of re-deciding every tick — without the hold it would jitter at the point where the shorter side flips.

The loop counts up to the move scan points but the success test compares against the *pounce* scan points, a larger number, so a full sweep that finds nothing still reports success with a stale target. In practice a full sweep failing means the enemy is standing somewhere unreachable and the caller's fallback covers it.

## `check_control_start_conditions` — the arbitration hook

**Contract** — Answers whether a named control ability may start right now. Allows the jump ability only while `allow_jump` is raised; permits everything else.

**Notes** — This is how the state keeps the shared jump ability from launching pounces on its own initiative. The ability is polled continuously by the control manager; without this latch a chimera would jump whenever the ability happened to like the geometry, rather than when the fight's rhythm called for it. The latch pattern — raise, ask, lower — appears again in the burer's anti-aim state, and it is the chapter's standard way of saying "this ability fires only when I say so".

## `finalize` / `critical_finalize`

**Contract** — Identical: if the state had taken path or movement control, give each back, and clear any override animation. Both are guarded by "only if this state is still the holder", since another ability may have taken over in the meantime.

**Invariants** — After either, the creature's path follower is driving again and no clip is being forced.
