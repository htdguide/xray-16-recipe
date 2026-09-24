# src/xrGame/ai/monsters/rats/rat_state_activation.cpp

> What each rat state actually does on a tick once it has decided to stay: wander, settle, charge, bite, bolt, go home, or eat.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`ai_rat_impl.h`](ai_rat_impl.h.md) · [`ai_rat_space.h`](ai_rat_space.h.md) · [`../../../memory_manager.h`](../../../memory_manager.h.md) · [`../../../item_manager.h`](../../../item_manager.h.md) · [`../../../sound_player.h`](../../../sound_player.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: per-state parameter assignment; the motion itself is elsewhere

## Purpose

Eight routines, one per state that has per-tick work. None of them moves the rat: they set the
goal, the speed and the two movement flags, and the integration in
[`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) does the rest on the frame update. That split
is the thing to notice — **the brain runs on the rate-degraded schedule and the locomotion runs
every frame**, which is what lets a nest of forty rats think a few times a second and still
move smoothly.

The two movement flags each routine sets are the whole vocabulary: *may adjust speed*, meaning
the rat is allowed to slow itself for turns and mesh edges, and *straight at goal*, meaning it
is charging a target rather than ambling toward one.

## `activate_state_free_active` — wandering

**Contract** — the default state. Anchors the wander box on the nest's home position, restores
the wandering turn rate and goal interval, and on each goal-interval edge either re-rolls the
speed or — if the rat is close to home and the dice say so and it can stand where it is —
*leaves the activity budget and settles*. Plays the idle chitter on a long random interval.

```text
FUNCTION activate_state_free_active()
  anchor        = home_position
  goal_delta    = authored wander interval
  goal_box      = authored wander variation
  turn_rate     = authored wander turn rate

  IF goal_countdown_expired()                    # re-arms and re-rolls the goal
    IF distance(me, anchor) > stable_distance OR random() > settle_probability
      IF distance(me, home) > home_radius
        speed = safe_speed = max_speed           # too far out: hurry
      ELSE
        choose_new_speed()
    ELSE IF I can stand where I am
      leave_activity_budget()                    # settle; the schedule slows down

  IF speed is zero AND my turn is nearly finished
    choose_new_speed()                           # don't stand still by accident

  tick_goal_countdown()
  movement_flags = { may_adjust_speed: false, straight_at_goal: false }
  play(idle_chitter, interval 45s, jitter 15s)
```

**Invariants**

- **Settling is gated on three things at once**: being near home, losing a dice roll, and not
  standing on another rat. The third is what keeps a settling nest from stacking, and it is
  checked *before* the budget is left, so a rat that cannot settle keeps wandering rather than
  freezing in place.
- **The "too far from home" branch is an alternative to the speed lottery, not a prelude to
  it** — unlike the equivalent in
  [`rat_state_initialize.cpp`](rat_state_initialize.cpp.md), where the same two lines fall
  through and the force is lost. This is the correct one.
- **The second speed check exists because the lottery may leave a rat at zero.** Without it a
  rat that drew a stop and finished its turn would stand still until its next goal interval.
- **Speed adjustment is disabled** while wandering, so the ladder in `select_speed` never runs
  and the rat travels at exactly the speed the lottery gave it. Wandering rats therefore do not
  slow for turns; they stop and turn, which is what the animation selector expects.

## `activate_state_free_passive` — settled

**Contract** — the cheap state. Checks three wake conditions and forces itself active on any of
them; otherwise stops dead, joins the standing budget if there is room, and asks (unforced) to
rejoin the active budget. Plays the same idle chitter.

```text
FUNCTION activate_state_free_passive()
  IF an enemy exists
    goal_countdown = 0                    # re-goal immediately on waking
    join_activity_budget(forced: true) ; RETURN
  IF morale < normal
    join_activity_budget(forced: true) ; RETURN
  IF a current sound was made by something not on my team
    join_activity_budget(forced: true) ; RETURN

  speed = 0
  join_standing_budget()                  # subject to its own proportion cap
  join_activity_budget(forced: false)     # subject to the active cap; usually refused
  play(idle_chitter, interval 45s, jitter 15s)
```

**Invariants**

- **The three wake conditions are forced and bypass the budget entirely.** An enemy, fear, or a
  hostile noise wakes every settled rat regardless of how many are already active. That is the
  nest's alarm, and it is the reason the budget throttles boredom rather than reaction.
- **The unforced rejoin at the end is the churn** that keeps a nest from ossifying: every
  settled rat asks to wake each time it runs, and the proportion cap admits a few. Combined with
  active rats settling in the routine above, a nest continuously exchanges which of its members
  are moving.
- **Clearing the goal countdown before waking on an enemy** makes the woken rat pick a fresh
  goal on its very first active tick rather than steering at a stale one.

## `activate_state_attack_range` — biting

**Contract** — stop, face-lock, and bite. Also runs a five-second timer whose only consumer is
the brain's own re-approach test.

```text
FUNCTION activate_state_attack_range()
  IF the rebuild timer is not running
    start it
  IF it has run five seconds
    stop it
  speed = 0
  fire(true)                              # arms the bite; the cooldown is enforced in Exec_Action
  movement_flags = { may_adjust_speed: false, straight_at_goal: false }
```

**Invariants** — the five-second window is a *grace period*: while it is running, the state that
tests it will not abandon the bite because another rat is standing in the way. It is what stops
a swarm from cycling between biting and shuffling. The timer starts on the first tick of biting
and expires once; it does not restart while the state continues.

**Notes** — the name says "range" and the behaviour is melee; see
[`ai_rat.h`](ai_rat.h.md) on the misleading state names.

## `activate_state_move` and `activate_move`

**Contract** — the two travelling steps. `activate_state_move` charges at attack speed;
`activate_move` runs at maximum speed with whichever flags the patrol logic set. Both tick the
goal countdown.

**Invariants** — charging sets *both* flags: speed may be adjusted (so the ladder engages and
the rat slows for tight turns rather than stopping) and it is travelling straight at its goal
(so the steering's dead band engages and it does not weave). Those two flags together are what
a "charge" is in this creature.

`activate_move` is the patrol step, and it takes its flags from the rat's own fields rather than
fixing them — which is the one place the rat's movement mode is data rather than code. Nothing
shipped writes those fields to anything but their defaults.

## `activate_turn`

**Contract** — points the rat's body directly at its enemy and sets the goal to the enemy's
position. Used by the states that must face a target before acting.

**Notes** — the direction is built by dividing the offset by the distance three times, once per
axis, rather than normalising once. Arithmetically identical, three times the divisions, and it
divides by zero if the rat is exactly on top of its enemy — which the mesh veto makes
practically impossible but does not forbid.

## `activate_state_free_recoil` — bolting

**Contract** — full speed, charging flags, idle chitter. The startle has no steering of its own:
the goal was placed ahead of the rat when the state was entered (see
[`rat_state_initialize.cpp`](rat_state_initialize.cpp.md)) and the rat simply runs at it.

## `activate_state_home` — going back

**Contract** — re-anchors on the nest, restores the wandering turn rate and goal interval, and
runs at attack speed with speed adjustment enabled and charging disabled.

**Invariants** — a returning rat runs at *attack* speed, not maximum, which is the fastest of the
four. Going home is the most urgent thing a rat does.

**Notes** — the speed is assigned, the countdown ticked, and then the speed assigned again to the
same value. Harmless duplication from the port.

## `activate_state_eat` — feeding

**Contract** — steer at the corpse's centre until close and roughly facing, then stop, take a
bite on the authored interval, and play the eating sound. If not yet in position, run at
maximum speed with charging flags and stop the eating sound.

```text
FUNCTION activate_state_eat()
  corpse_centre = centre of the selected corpse
  IF the goal has not been refreshed for 2 seconds
    goal = corpse_centre
  tick_goal_countdown()

  in_reach = distance(corpse_centre, me) <= attack_distance
  facing   = angle between my heading and the direction to the corpse < 30 degrees

  IF in_reach AND facing
    speed = 0
    IF now - last_bite > hit_interval
      last_bite = now
      corpse.remaining_food = corpse.remaining_food - hit_power / 10
    biting = true
    movement_flags = { may_adjust_speed: false, straight_at_goal: false }
    play(eating_sound)
  ELSE
    stop all sounds
    speed = IF in_reach THEN 0 ELSE max_speed     # in reach but not facing: stop and turn
    movement_flags = { may_adjust_speed: true, straight_at_goal: true }
```

**Invariants**

- **Eating consumes a tenth of the rat's bite damage per interval.** That ratio is compiled in
  and is what sets how long a nest takes to strip a body; the authored bite damage therefore
  tunes both combat and feeding at once, which a rebuild separating them should be aware of.
- **In reach but not facing means stop, not circle.** Speed goes to zero and the steering turns
  the rat in place, which is why a feeding rat snaps round to its carcass rather than orbiting.
- **The eating sound is stopped on every tick the rat is not actually biting**, including the
  approach, so the sound tracks the act rather than the state.
- The goal is refreshed only every two seconds rather than every tick, which is enough for a
  corpse (which does not move) and saves the steering from re-aiming at sub-unit jitter.

**Notes** — the two-second goal refresh interval and the thirty-degree facing tolerance are
compiled in, as is the tenth. The same block appears almost verbatim in the dead brain
([`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md)), which is the clearest surviving evidence of how the
port was done.
