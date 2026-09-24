# src/xrGame/ai/monsters/pseudodog/psy_dog_state_psy_attack_hide_inline.h

> The psi dog's retreat: pick a cover point that hides you from where the enemy is, run there at full aggression, and stop when you are standing on it.

**Needs** — [`psy_dog_state_psy_attack_hide.h`](psy_dog_state_psy_attack_hide.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../../../cover_point.h`](../../../cover_point.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`psy_dog_state_psy_attack_hide.h`](psy_dog_state_psy_attack_hide.h.md)
**Tier floor** — T3: a cover query and a path request

## Purpose

The psi dog's whole contribution to a fight is staying alive behind its phantoms; this is the
move that does it. The interesting parts are the *two-stage* cover choice and the fact that
the state loops rather than terminating — see
[`psy_dog_state_psy_attack_inline.h`](psy_dog_state_psy_attack_inline.h.md).

## `select_target_point`

**Contract** — chooses the destination, once, on entry. Never fails: it always produces a
vertex and position, even when no cover exists.

```text
FUNCTION select_target_point()
  # first choice: cover measured against where the enemy is
  point = cover_manager.find_cover(from: enemy_position, min_radius: 10, max_radius: 30)
  IF point EXISTS AND distance(self, point) > 2
    target = point
    RETURN

  # second choice: cover measured against where I myself am
  point = cover_manager.find_cover(from: self_position, min_radius: 10, max_radius: 30)
  IF point EXISTS AND distance(self, point) > 2
    target = point
    RETURN

  # no cover at all
  target.vertex = 0
  target.position = position_of_mesh_vertex(0)
```

**Invariants**

- **The two-metre floor is what makes the move a move.** A cover point the dog is already
  standing on would complete instantly and the dog would re-pick every tick; rejecting anything
  closer than two world units forces genuine travel.
- **The search annulus is ten to thirty world units** in both stages, so the dog never hides
  right next to the enemy and never runs off the map to do it.
- **The two stages ask different questions.** The first means "somewhere the enemy cannot see
  me"; the second, used when the first finds nothing, means "somewhere near here that is
  covered from *something*" — a weaker guarantee, taken because moving is better than standing
  still in the open.

**Notes** — the final fallback is the bug on this page: with no cover found, the dog is sent to
**mesh vertex zero**, which is not a considered location but simply the first vertex in the
level's navigation mesh — an arbitrary corner of the map. A psi dog that fails both cover
searches sprints across the entire level. A rebuild should fall back to the dog's current
position (completing immediately and letting the parent re-select) or to the point furthest
from the enemy among those already examined.

## `initialize`

**Contract** — picks the destination and tells the path builder a new request is coming, so
the previous route is discarded rather than extended.

## `execute`

**Contract** — one tick. Sets the running action, re-asserts the path request, drives the
animation accelerator into its aggressive profile with braking off, and plays the aggressive
vocalisation on the creature's authored attack-sound interval.

```text
FUNCTION execute()
  set_action(run)
  path.target = target                 # re-asserted every tick, which is how the builder learns
  path.rebuild_interval = 0            # rebuild whenever the route is invalidated, no throttle
  path.stop_distance = 0               # go all the way to the vertex
  path.use_covers = false              # the destination IS the cover; don't route through more
  animation.accelerate(aggressive, braking: false)
  sound.play(aggressive, delay: authored attack-sound interval)
```

**Invariants** — braking is explicitly disabled so the dog arrives at full speed instead of
decelerating into the cover, and cover-seeking is explicitly disabled along the route, because
routing through cover on the way to cover would make the approach wander. Both are the
distinctive settings; everything else is the chapter's standard run.

## `check_completion`

**Contract** — true when the dog occupies the destination vertex *and* the path builder reports
it is no longer moving. Both halves are needed: occupying the vertex while still sliding means
the arrival animation has not settled, and stopping short of the vertex means the route
failed rather than finished.

## `check_start_conditions`

**Contract** — always true. The state is selected unconditionally by its parent, which has
already decided the dog must hide; there is no second opinion here.
