# src/xrGame/ai/monsters/rats/rat_state_switch.cpp

> The rat's vocabulary of questions and small actions: every condition a state may branch on, and every one-line thing a state may do, each named and exposed so that the states themselves are almost free of logic.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`ai_rat_impl.h`](ai_rat_impl.h.md) · [`../../../memory_manager.h`](../../../memory_manager.h.md) · [`../../../enemy_manager.h`](../../../enemy_manager.h.md) · [`../../../item_manager.h`](../../../item_manager.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../../ai_monsters_misc.h`](../../ai_monsters_misc.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: predicates and setters over the rat's own fields

## Purpose

When the rat's brain was rebuilt from the switch loop in
[`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md) into a stack of state objects, every condition and every
action inside the old switch arms was lifted out and given a name. This file is the result: a
flat vocabulary that the state objects compose. That is the only reason it exists, and it is a
good reason — it makes the states readable and it makes the conditions testable in isolation.

The cost is that several of the names are poor and a few of the predicates are compound
conditions with no single meaning. Where a name misleads, this page says what the predicate
actually tests.

## The questions about an enemy

**`switch_if_enemy`** — is there a selected enemy at all.

**`switch_if_alife`** — is that enemy still alive. **Assumes one exists**; every caller checks
first.

**`switch_if_position`** — is the enemy further away than the rat's bite reach.

**`switch_if_diff`** — is the rat facing more than its authored attack angle away from the
enemy.

**`switch_if_dist_angle`** — in reach *and* facing: the bite is on. The composition of the two
above, negated.

**`switch_if_dist_no_angle`** — in reach but *not* facing. **Has a side effect**: it sets the
rat's speed to zero, so asking the question stops the rat. That is deliberate — a rat that is
close enough but turned away should be pivoting, not travelling — but a predicate that moves
the creature is bad shape and a rebuild should make it an action.

**`switch_if_porsuit`** *(sic)* — is the enemy beyond the nest's pursuit radius, measured from
the **home anchor**, not from the rat. A rat therefore gives up on an enemy that leaves the
nest's territory even if the rat itself has chased it out.

**`switch_if_home`** — is the rat still inside its own home radius.

**`switch_to_attack_melee`** — may the rat commit to a charge.

```text
FUNCTION switch_to_attack_melee() -> bool
  RETURN (the enemy is inside the nest's pursuit radius) OR (I have strayed outside my home radius)
```

**Invariants** — the disjunction is the whole territorial model and is easy to misread from the
negated form the source uses. The first clause is the normal case: fight what comes into the
nest. The second is the escape valve: a rat that has already wandered too far fights whatever
it meets, because it has no territory left to defend. Without it, a strayed rat would refuse to
fight and would also refuse to flee.

**`switch_if_lost_time`** — has the enemy gone unseen longer than the authored memory time; if
so, **forgets it** as a side effect. Used to end a pursuit.

**`switch_if_no_enemy`** — the compound "stop retreating" test.

```text
FUNCTION switch_if_no_enemy() -> bool
  IF no enemy
     OR the enemy is dead
     OR (unseen longer than the retreat time
         AND (the recorded sound is stale OR has no source
              OR the source is on my team OR the sound was not gunfire))
    forget the enemy
    RETURN true
  RETURN false
```

**Invariants** — the inner clause is what keeps a rat retreating while shooting continues. Time
alone does not end a retreat; the rat must also stop hearing hostile gunfire. A rebuild that
drops the sound clause gets rats that turn and charge into a firefight on a timer.

**`switch_to_free_recoil`** — the startle test.

```text
FUNCTION switch_to_free_recoil() -> bool
  RETURN a current sound exists
     AND (it has no source
          OR ((it did not come from the corpse I am eating) AND (its source is not on my team)))
     AND I have no enemy
```

**Invariants** — the exclusion of the corpse being eaten is the load-bearing clause: without
it, the noise a rat makes while feeding startles the rat that is feeding, and a nest at a
carcass scatters itself.

**`switch_if_lost_rtime`** — the startle's expiry, comparing the enemy's last-seen time against
the startle stamp. **Assumes an enemy exists** and is called on paths where one may not; see the
note below.

**`switch_if_time`** — has the startle been running more than two seconds. The two seconds is
compiled in.

## The questions about the rat itself

**`get_alife`** — is the rat alive. **Has a side effect**: a dead rat's speed is zeroed. Every
state's tick begins with this, which is how a corpse is stopped exactly once.

**`get_morale`** — is morale at or above normal, within an epsilon. The single threshold that
separates fighting from fleeing.

**`get_if_dw_time`** — is the recorded sound current rather than stale. Named for the field it
reads; what it means is "I have heard something since I last ran".

**`get_if_tp_entity`** — was the recorded sound made by a hostile that was *not* frightening —
that is, by something on another team, and not gunfire. This is the "something is moving over
there" test, as opposed to "someone is shooting", and it is what turns a rat toward a noise
rather than away from it.

**`switch_to_eat`** — has the item system selected a corpse for this rat.

**`check_completion_no_way`** — has the bite cooldown elapsed. Named for the patrol state it was
moved into and tests nothing about paths.

**`get_state`** — the group combat lottery. Calls a shared group-level decision routine with the
rat's team, squad and group, the authored success probability and refresh rate, and two
candidate outcomes — charge or retreat — and returns which the group has settled on.

**Invariants** — the point is that the *group* decides, not the rat: the routine is keyed on the
hierarchy identifiers and a refresh rate, so every rat in a group asking within the same window
gets the same answer. A nest therefore commits to a charge or breaks together, which is what
makes a swarm feel like a swarm. All four probability arguments are passed the same authored
value, so the lottery is effectively a single-probability coin; the routine's shape allows four
different ones and the rat does not use them.

## The actions

**`set_dir`** — point the goal at the enemy, **fanned out by squad position**.

```text
FUNCTION set_dir()
  IF the goal was refreshed less than 2 seconds ago  RETURN
  target = enemy position
  IF my squad is active
    step   = a full turn divided by the number of living squad members
    bearing = bearing from the enemy to the squad leader, rotated by step * my index
    target = target + (unit vector along that bearing) * 0.5
  goal = target
```

**Invariants** — this is the rat's entire flanking behaviour and it is four lines. Each rat is
given a distinct slot on a half-unit circle around the enemy, indexed by its position in the
squad and rotated so that slot zero is where the leader approaches from. The result is that a
swarm surrounds rather than queues. The offset is deliberately tiny — half a unit — because it
only needs to break the tie between identical approach vectors; the mesh veto and the rats'
mutual standing test do the rest of the spacing.

The squad indices are refreshed only when the living count changes (see
[`ai_rat.cpp`](ai_rat.cpp.md)), so the fan collapses and re-forms as rats die.

**`set_dir_m`** — point the goal at the enemy's *remembered* position rather than its actual one.
The pursuit step: a rat chasing something it cannot see runs to where it last was.

**`set_sp_dir`** — point the goal at the flee anchor.

**`set_home_pos`** — point the goal at the nest anchor.

**`set_way_point`** — point the goal at the next patrol waypoint, advancing it if reached.

All four share a **two-second refresh throttle** except `set_way_point`, which is unthrottled
because a waypoint must be re-read the moment the previous one is passed.

**`set_rew_position`** — place the flee anchor one authored retreat-distance directly away from
the enemy.

**Notes** — the routine computes the direction *toward* the enemy first, then immediately
overwrites it with the direction away. The first computation is dead; only the second matters.

**`set_rew_cur_position`** — place the flee anchor one authored flee-distance directly *ahead*
of the rat's current facing. The distinction from the above is the one that matters: retreating
backs away from a known enemy, while being frightened runs forward from nothing in particular.

**`set_movement_type`** — set the two movement flags the locomotion reads.

**`set_firing`** — set the biting flag without arming the attack action, unlike `fire`. Used to
stop biting without emitting an attack-end action.

**`set_previous_query_time`** — stamp the throttle clock, so the next goal-setting call is
allowed through.

**`set_goal_time`** — clear the goal countdown so the next tick re-rolls. **Takes a value and
ignores it**, always assigning zero. Every caller passes the default. Dead parameter.

**`sub_rotation`** — the bearing from the rat to its enemy, as an orientation. A helper for the
facing predicates above.

**`get_enemy`** — the selected enemy.

## Notes on shape

Several predicates dereference the enemy without checking one exists, and are safe only because
of the order the states call them in. `switch_if_lost_rtime` and `switch_if_position` are the
two that would fault first if a state's ordering changed. A rebuild should make the enemy an
optional value and let the predicates answer false when it is absent, which costs nothing and
removes an entire class of ordering constraint that is currently invisible in the source.
