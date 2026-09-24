# src/xrGame/ai/monsters/zombie/zombie_state_attack_run_inline.h

> Implements the shamble: walk at the enemy, re-pathing more lazily the further away he is, and break into a run only after being shot.

**Needs** — [`zombie_state_attack_run.h`](zombie_state_attack_run.h.md) · [`ai_monster_squad.h`](../ai_monster_squad.h.md) · [`sound_player.h`](../../../sound_player.h.md)
**Used by** — [`zombie_state_attack_run.h`](zombie_state_attack_run.h.md)
**Tier floor** — T3: a path request with a distance-scaled cadence, plus a one-line gait rule

## Purpose

Two decisions here, and the first is the one a rebuild will feel.

**The re-path cadence scales with distance to the enemy.** A zombie a hundred metres away
recomputes its route every five and a bit seconds; a zombie two metres away recomputes it
ten times a second. This is the creature layer's basic load-shedding idiom: the cost of
being wrong about a route is proportional to how close you are, so pay for accuracy only
when it shows. Every rebuild of a creature approach wants this shape.

**The zombie shambles until provoked.** Its gait is walking unless its hit memory says it
has been hit, and then it runs. Not "runs while hurt" — runs while *remembering* being hit,
which is a decaying memory, so a zombie that takes a shot lurches into a run and drops back
to a walk when the memory lapses.

## `CStateZombieAttackRun`

**Contract** — entry resets the gait to walking and prepares the path builder. Each tick it
restates the destination as the enemy's remembered position and navigation vertex, sets the
re-path cadence, sets a two-and-a-half-metre stop distance, turns cover preference **off**,
applies the squad's assigned facing if the zombie is in an active squad under an attack
order, picks the gait, vocalises aggression, and engages the aggressive acceleration profile
without braking.

The start condition and the completion test are the two ends of melee range, read from the
creature's melee component: start when the enemy is beyond the maximum, finish when he is
inside the minimum. That is what hands control back to the strike phase of the attack
composite.

```text
FUNCTION initialize()
  action = walk_forward
  object.path.prepare_builder()

FUNCTION execute()
  distance = enemy.position distance to object.position

  object.path.set_try_min_time(false)          # default: prefer a safe route over a fast one
  object.path.set_target_point(memory.enemy.position, memory.enemy.vertex)

  # the cadence: 100 ms plus 50 ms per metre of separation.
  # 2 m -> 200 ms; 20 m -> 1.1 s; 100 m -> 5.1 s.
  object.path.set_rebuild_time(100 + 50 * distance)

  object.path.set_distance_to_end(2.5)
  object.path.set_use_covers(false)            # a zombie does not flank

  squad   = squad_registry.squad_of(object)
  command = squad.command_for(object)
  IF squad EXISTS AND squad.is_active AND command.type == attack
    object.path.set_use_dest_orient(true)
    object.path.set_dest_direction(command.direction)   # the squad assigns an approach bearing
  ELSE
    object.path.set_use_dest_orient(false)

  choose_action()
  object.animation.action = action
  IF action == run
    object.path.set_try_min_time(true)         # once running, take the fastest route

  object.sound.play(aggressive, delay = creature's configured attack sound delay)
  object.animation.acceleration_activate(aggressive)
  object.animation.acceleration_set_braking(false)

FUNCTION check_start_conditions() -> bool
  RETURN melee.distance_to(enemy) > melee.max_distance

FUNCTION check_completion() -> bool
  RETURN melee.distance_to(enemy) < melee.min_distance

FUNCTION choose_action()
  action = memory.hits.was_hit ? run : walk_forward
```

**Invariants**

- Route *style* follows gait: walking takes the safe route, running takes the fast one. The
  flag is cleared at the top of every tick and set again only for the running case, so the
  two never drift apart.
- Start condition and completion test use different thresholds — maximum to start, minimum
  to finish — which gives the approach hysteresis. Without it a zombie hovering at exactly
  melee range would oscillate between approaching and striking every tick.

## Notes

**A null dereference waiting on a squad.** The squad's order is fetched *before* the squad
handle is checked for existence; the check that follows only decides whether to *use* the
order. A zombie with no squad reaches the fetch through a missing handle. It survives in
practice because zombies are always spawned into a squad, which is an invariant of the
spawn data rather than of the code. A rebuild must not rely on that.

**A disabled gait rule, three ways.** The live gait choice is one line, marked "for test",
and three richer rules sit commented out beside it: a rank test that would exempt strong
creatures, a ten-second hold that would stop a zombie dropping straight back to a walk after
one shot, and a health test that would require the zombie to be below half health as well as
hit. The `time_action_changed` field exists only for the hold rule and is written nowhere in
the compiled path. The shipped zombie has the one-line rule; a rebuilder should implement
that and leave the rest.
