# src/xrGame/ai/monsters/states/monster_state_controlled_follow_inline.h

> Escorting: walk to a random point near the thing you are following, then rest for a random few seconds, with the threshold between the two re-rolled every time — so a pack of puppets mills around its owner instead of standing in a ring.

**Needs** — [`monster_state_controlled_follow.h`](monster_state_controlled_follow.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md)
**Used by** — [`monster_state_controlled_follow.h`](monster_state_controlled_follow.h.md)
**Tier floor** — T3: a randomised threshold and a restriction-aware destination

## Purpose

Two generic children and twenty lines of parameterisation, with two decisions that matter: the
*randomised* follow threshold, and the fact that the walk aims at a random point near the
target rather than at the target itself.

## `reselect_state`

**Contract** — chooses between waiting and walking, by comparing the distance to the followed
object against **a threshold drawn fresh from a range on every selection**.

```text
FUNCTION reselect_state()
  target = my controlled-entity facet's object
  d      = distance(me, target)
  select(d < uniform_real(2, 10) ? wait : walk_to_object)
```

**Invariants** — this is the page's real idea. A fixed threshold makes every follower hold
exactly one distance and a group of them forms a visible ring; a threshold re-drawn each time
makes each follower's stopping distance vary between two and ten units from moment to moment,
so the group spreads unevenly and keeps shifting. Below two units a follower always waits, above
ten it always walks, and in between it is genuinely undecided — which is what produces the
milling.

The selector runs only when the previous child has completed, so the re-roll happens once per
wait or walk, not once per tick.

## `setup_substates` — the wait

**Contract** — parameterises the waiting child: the resting action, the creature's authored idle
vocalisation and its authored idle interval, and a timeout drawn between four and six seconds.

**Invariants** — the wait's *duration* is also randomised, independently of the threshold. Two
independent random quantities per cycle is what keeps a group of followers from ever falling
into step.

## `setup_substates` — the walk

**Contract** — parameterises the walking child: a destination sampled at random within ten units
of the followed object, adjusted if it lies outside the creature's movement restrictions;
walking rather than running; calm acceleration with braking off; completion at the inner
distance; the idle vocalisation; and **no route rebuilding at all**.

```text
FUNCTION setup_walk()
  destination = random_position_within(target.position, 10)
  IF destination is outside my movement restrictions
    destination = nearest accessible point to it, with its vertex
  ELSE
    destination stands as given, vertex left for the builder to resolve

  action           = walk
  acceleration     = calm, braking off
  completion_dist  = 2 units
  sound            = idle at the authored idle interval
  rebuild_interval = never
```

**Invariants**

- **The destination is near the target, not the target.** Escorts converge on a neighbourhood
  rather than on a point, which is the second half of the anti-ring design.
- **The restriction check runs before the walk begins**, and produces both a position and a
  vertex when it has to correct. Skipping it would let a puppet be ordered into a volume it is
  forbidden to enter, where the pathfinder would refuse and the walk would never complete.
- **No rebuilding.** The route is planned once. Since the target is moving, the destination goes
  stale almost immediately — which is fine, because the walk completes within two units of a
  point that was near the target a few seconds ago, and the selector then re-rolls. Following is
  achieved by *repeated short stale walks*, not by tracking. A rebuild that replans continuously
  gets a puppet glued to its owner's back, which is a different and worse behaviour.
- **Walking, not running**, and calm acceleration: an escorting creature is deliberately slower
  than its owner, so a moving controller strings its puppets out behind it.
