# src/xrGame/ai/monsters/bloodsucker/bloodsucker_attack_state_inline.h

> The combat the bloodsucker was designed to fight: feed when the chance comes, vanish and circle whenever a chunk of health is lost, and close on the enemy's back rather than his front. Unreachable in the shipped build.

**Needs** — [`bloodsucker_attack_state.h`](bloodsucker_attack_state.h.md) · [`state_move_to_point.h`](../states/state_move_to_point.h.md) · [`monster_state_attack.h`](../states/monster_state_attack.h.md) · [`bloodsucker_vampire_execute.h`](bloodsucker_vampire_execute.h.md) · [`bloodsucker.h`](bloodsucker.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`monster_home.h`](../monster_home.h.md) · [`state.h`](../state.h.md)
**Used by** — [`bloodsucker_attack_state.h`](bloodsucker_attack_state.h.md)
**Tier floor** — T3: selection over perception facts plus path parameters; no device or format contact

## Purpose

Two behaviours live here. The first replaces the shared attack composite's substate selector with one that inserts *feeding* near the top and *withdraw-and-cloak* in the middle. The second is the withdrawal's approach leaf: a run at the enemy that can be told to arrive matching his facing, so the creature comes out of the cloak behind him.

Neither is reachable: the creature's brain registers the generic attack composite, and the line that would register this one is commented out beside it. The recipe keeps the page because the selector below is the clearest statement of the creature's intended combat identity, and because the back-approach state is not duplicated anywhere else.

## State

```text
RECORD BloodsuckerAttack                 # extends the shared attack composite
  time_stop_cloak  : int            # written on entry, never read
  dir_point        : vector         # declared, never used
  last_health      : real           # health when the last withdrawal decision was taken
  start_with_circle: bool           # the next withdrawal should begin circling

RECORD BackstabEnemy                     # a leaf approach state
  params.start_with_circle : bool   # handed in by the composite above
  last_health          : real
  circling             : bool
  circle_end_tick      : int
  next_flip_allowed_at : int
```

Three tuned constants, fixed in code rather than read from the creature's section: a circling
burst lasts 3 seconds, a "meaningful wound" is 15 percent of maximum health, and behaviour may
flip at most once per second.

## `BloodsuckerAttackState`

**Contract** — on entry, clear the cloak stamp and record current health. On either exit, put the creature back into its stalking cloak. Each update, pick exactly one substate by the ordered test below, execute it, remember it as previous, and publish the creature's goal to its pack.

**Invariants** — the creature always leaves this composite cloaked, by either exit path, so combat ending never leaves a bloodsucker standing visible. While melee is the chosen substate the "run away while invisible" request is cleared, so the creature commits to the strike instead of fading mid-swing.

```text
FUNCTION execute()
  IF enemy is outside the home region      -> move_to_home_point
  ELSE IF feeding is wanted (see below)    -> vampire_execute
  ELSE IF the shared steal test passes     -> steal
  ELSE IF the shared camp test passes      -> camp
  ELSE IF the shared lost-enemy test passes-> find_enemy
  ELSE IF withdrawal is wanted (see below) -> withdraw
  ELSE IF the shared run-attack test passes-> run_attack
  ELSE
    melee_wanted = (previous was melee AND melee has not completed)
                   OR (previous was not melee AND melee will accept)
    IF previous was melee AND NOT melee_wanted -> withdraw    # never stand still after a strike
    ELSE IF melee_wanted                       -> melee
    ELSE                                       -> run_at_enemy

  IF chosen is not melee
    clear the shared "enemy behind me" timer
  ELSE
    clear the creature's run-away-invisible request

  run(chosen)
  previous = chosen
  tell the pack: my goal is "attack this enemy"
```

**Notes** — the "previous was melee and melee no longer wants to run" branch is the rhythm of the whole fight: a bloodsucker never finishes a strike and stands there, it always breaks contact. That single line is what makes the creature read as a stalker rather than a brawler.

A disabled branch beside the melee choice would have diverted to a run-away state when the enemy had been behind the creature for a while; the note left with it says it caused rotation snapping and wants its own state. The shared "enemy behind me" timer it read is still maintained.

## `check_feeding` and `check_withdrawal`

**Contract** — both answer "should this composite switch to the corresponding substate", and both use the same latch shape: if we are not already in it, ask whether it will start; if we are, ask whether it has *not* finished. That shape is what stops a substate being re-entered the tick after it completes.

```text
FUNCTION check_feeding() -> bool
  IF previous is not vampire_execute  RETURN vampire_execute.will_start()
  RETURN NOT vampire_execute.is_complete()

FUNCTION check_withdrawal() -> bool
  IF health < last_health - 0.15
    request run-away-while-invisible
    last_health = health
    start_with_circle = true
    RETURN true

  IF a critical hit landed within the last second
    consume that critical-hit mark
    start_with_circle = true
    RETURN true

  IF withdrawal is the active substate   RETURN NOT active.is_complete()

  start_with_circle = false
  RETURN withdrawal.will_start()
```

**Notes** — the two entry paths differ only in what they remember: a health-step loss updates the baseline so the *next* 15 percent is measured from here, while a critical hit clears its mark so it fires once. Both set the circling flag, so a wound is what makes the creature orbit rather than simply retreat. Withdrawal entered by the ordinary route (the enemy is merely too far for melee) runs straight.

## `setup_substates`

**Contract** — when the chosen substate is the withdrawal, fill its parameter record: run, no timeout, arrive within one unit, rebuild the path every 200 milliseconds, aggressive acceleration without braking, idle vocalisation at the creature's configured idle delay, and the circling flag decided above. Every other substate is left to the shared composite.

## `BackstabEnemyState`

**Contract** — on entry, prime the path builder, record health, adopt the circling flag handed in, set the circling burst to expire 3 seconds out. Each update, possibly flip between circling and straight pursuit, then re-issue the movement request and the path parameters. It will start only when the enemy is inside the creature's home region and further away than the melee reach; it completes when the enemy leaves the home region, or when the path has genuinely ended — and, when circling, only if the creature can see the enemy while the enemy cannot see it.

**Invariants** — the target point is re-read from the enemy every update, so the route tracks a moving target. Circling and straight pursuit are mutually exclusive: circling asks the path builder for a destination *orientation* and gives up on minimising travel time; straight pursuit asks for minimum time and no orientation constraint.

```text
FUNCTION execute()
  IF health < last_health - 0.15 AND now > next_flip_allowed_at
    next_flip_allowed_at = now + 1 second
    last_health = health
    circling = NOT circling                      # being shot changes the approach
    IF circling  circle_end_tick = now + 3 seconds

  IF now > circle_end_tick AND enemy can see me
    circling = false                             # circling has stopped paying

  request_action(run)
  target = enemy position                        # vertex left unresolved, see Notes
  desired_facing = enemy's own facing            # arrive pointing where he points

  path.target        = target
  path.rebuild_every = params.time_to_rebuild
  path.stop_distance = params.completion_dist
  path.use_covers    = true
  path.cover_params  = (5, 30, 1, 30)

  IF circling
    path.destination_orientation = desired_facing
    path.minimise_travel_time    = false
  ELSE
    path.minimise_travel_time    = true
    path.destination_orientation = none

  apply acceleration profile and vocalisation from params

FUNCTION check_completion() -> bool
  IF enemy is outside the home region  RETURN true
  arrived = path.is_at_end(params.completion_dist)
            AND (params.completion_dist > 0
                 OR horizontal distance to target < one navigation cell)
  IF NOT arrived  RETURN false
  IF NOT circling  RETURN true
  RETURN (I can see the enemy) AND (the enemy cannot see me)
```

**Notes** — asking the path builder to *arrive facing the enemy's facing* is the whole trick: a route that ends pointing the same way the enemy points is a route that ends behind him. The cost of that constraint is travel time, which is why it is given up as soon as the enemy has spotted the creature anyway — circling a target that is watching you is wasted motion.

The target vertex is set to zero rather than resolved from the enemy's position, so the path request carries a position with a wrong vertex attached. The path builder recovers by resolving the position itself; a rebuild should resolve it here and not rely on that.

The completion test's extra "within one navigation cell" clause exists because a zero arrival distance is not achievable exactly — the path builder reports path-end while the creature is still a cell away. The clause turns "the route ran out" into "I am actually there".

Its start condition consults the melee reach rather than a number of its own, so the creature only tries to circle when it is *not* already in striking range. Enemies inside melee reach are handled by the composite's melee branch.
