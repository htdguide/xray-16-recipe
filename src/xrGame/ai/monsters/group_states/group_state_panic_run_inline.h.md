# src/xrGame/ai/monsters/group_states/group_state_panic_run_inline.h

> Flee *toward* the pack's territory, on the far side from the enemy — and keep fleeing until you
> are both far away and unobserved.

**Needs** — [`group_state_panic_run.h`](group_state_panic_run.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`group_state_panic_run.h`](group_state_panic_run.h.md)
**Tier floor** — T3: one direction computation and a two-part predicate

## Purpose

The pack's answer to "run away". A solitary creature flees *from* a point; a pack creature flees
*toward* its home region, choosing the spot in the outer ring that lies on the far side from the
enemy. The difference is small in code and large in play: packs retreat as a body into familiar
ground and re-form there, rather than dispersing in every direction and never returning.

## State

`Stateless.` Two authored constants, both in code: a 15-second unseen interval and a 15-unit
minimum separation.

## `CStateGroupPanicRun`

**Contract** — prime the path builder on entry. Each tick request the run action, the panic sound,
and the aggressive acceleration profile without braking, then compute the destination afresh and
hand it to the path builder with the generic parameters. Finishes only when both safety conditions
hold.

```text
FUNCTION execute()
  request_action(run)
  set_state_sound(panic)
  animation.accelerate(aggressive, braking = false)

  direction = normalize(home_point - enemy_position)
  path.target = home.a_place_in_the_outer_region_toward(direction)
  path.set_generic_parameters()

FUNCTION is_finished() -> bool
  IF distance(self, enemy_position) < MIN_DIST_TO_ENEMY   RETURN false
  IF time_since_enemy_last_seen < MIN_UNSEEN_TIME         RETURN false
  RETURN true
```

**Notes** — the destination is recomputed **every tick**, not latched on entry, because the enemy
moves: a pursued creature continuously re-picks the spot in its territory furthest from where the
pursuer now is, which produces a curving flight rather than a straight line to a stale point. That
is one line and it is the difference between a pack that can be cornered and one that circles.

The direction is taken from the enemy *toward home*, which means the chosen spot is on the home
region's far edge as seen from the enemy — the same idiom
[`group_state_hear_danger_sound_inline.h`](group_state_hear_danger_sound_inline.h.md) uses for a
sound. Home is treated as a shape to hide behind, not a point to reach.

The finish test is a **conjunction**, and both halves are necessary for different reasons.
Distance alone would let a creature stop while still in the enemy's sights across open ground;
unseen-time alone would let a creature stop the moment it rounds a corner three metres away. The
fifteen-second interval is long — long enough that a fleeing pack will usually cross its whole
territory — and it is what makes panic in this game feel like a rout rather than a flinch.

Neither number is in data. The panic *sound* is, through the sound category; the timings are not.
