# src/xrGame/ai/monsters/controller/controller_state_attack_inline.h

> The controller's attack brain: defend the home region first, then alternate between closing on
> the enemy and biting, and stand and turn when the enemy is somewhere the creature cannot reach.

**Needs** — [`controller_state_attack.h`](controller_state_attack.h.md) · [`controller_state_attack_hide.h`](controller_state_attack_hide.h.md) · [`controller_state_attack_hide_lite.h`](controller_state_attack_hide_lite.h.md) · [`controller_state_attack_moveout.h`](controller_state_attack_moveout.h.md) · [`controller_state_attack_camp.h`](controller_state_attack_camp.h.md) · [`controller_state_attack_fire.h`](controller_state_attack_fire.h.md) · [`controller_tube.h`](controller_tube.h.md) · [`../states/monster_state_attack_run.h`](../states/monster_state_attack_run.h.md) · [`../states/monster_state_attack_melee.h`](../states/monster_state_attack_melee.h.md) · [`../states/monster_state_home_point_attack.h`](../states/monster_state_home_point_attack.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_state_attack.h`](controller_state_attack.h.md)
**Tier floor** — T3: a selector over three substates plus one geometric special case

## Purpose

A composite state: it owns three leaf states and chooses between them every tick. It is the
controller's answer to "an enemy exists", and it is deliberately *thin* compared to the
group-creature attack brain — the controller has no pack, so there is no squad coordination, no
encirclement and no stealth approach here. What it adds over the generic attack composite is one
case: what to do when the enemy is standing somewhere the navigation mesh cannot take the
creature.

## State

The composite owns no fields of its own beyond the substate bookkeeping every composite state
has (which substate is active, which was active last tick). Its three substates are the generic
ones: **move to home point**, **run at the enemy** and **melee**.

## `CStateControllerAttack`

**Contract** — each tick, clear any animation override left by the previous tick, then select and
run exactly one substate. Selection is a fixed priority: home-point defence wins; otherwise melee
if the melee state says it can start (or says it has not finished); otherwise run. The
inaccessible-enemy case pre-empts the run branch entirely.

**Invariants** — the previous-substate record is updated on every exit path, including the early
return in the inaccessible-enemy branch, because the home-point and melee tests both read it.
When the inaccessible branch fires it deliberately *clears* the active substate, so the next tick
re-selects from scratch rather than resuming whatever was running.

```text
FUNCTION execute()
  clear_animation_override()

  IF should_defend_home_point()
    run_substate(move_to_home_point)
    RETURN

  IF active_substate == melee
    next = melee_finished() ? run : melee
  ELSE
    next = melee_may_start() ? melee : run

  IF next == run AND NOT enemy_is_reachable()
    # the enemy stands where the navigation mesh cannot follow — do not run at it
    active_substate = none            # force a fresh choice next tick
    IF angle_between(self.facing, direction_to(enemy)) > 30 degrees
      play_turn_animation(toward the enemy's side)
      face(enemy)
    request_action(stand_idle)
    RETURN

  run_substate(next)

FUNCTION should_defend_home_point() -> bool
  # the same latch every attack composite uses: once we are going home, keep going home
  IF previous_substate was not move_to_home_point
    RETURN move_to_home_point.may_start()
  RETURN NOT move_to_home_point.is_finished()
```

**Notes** — the inaccessible-enemy branch is the only load-bearing thing in the file, and it
solves a visible problem: an enemy on a roof or behind a fence is at a position with no reachable
navigation vertex, and a creature that keeps requesting "run at it" spins in place or presses
into geometry. The response is to stand still and *turn* toward the enemy, playing an explicit
turn animation only when the misalignment exceeds 30 degrees — below that the turn animation
would be a twitch, so the facing request alone is used. The state stays available for a melee
switch the instant the enemy comes back down.

The constructor names three leaf states and the file's includes name six more. Those six —
retreat, retreat-lite, move-out-of-cover, camp, psychic fire and the psychic-attack tube — are
pulled in and never instantiated. Two percentages sit at the top of the file, apparently the
intended split between firing and the tube attack; nothing reads them. The honest reading is that
this composite was planned to be the controller's full ranged-combat brain and shipped as a
melee-and-close brain with the ranged half left in the tree.

The home-point latch is the same shape used by every creature's attack composite: once the
creature has decided to go home it re-decides "am I still going home" rather than "should I go
home", which stops it oscillating at the boundary of its home region.
