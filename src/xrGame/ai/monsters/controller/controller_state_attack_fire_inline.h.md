# src/xrGame/ai/monsters/controller/controller_state_attack_fire_inline.h

> The controller's stand-still psychic-fire state: freeze, stare the enemy down at range, and
> let the psy attack fire with its cooldown suppressed.

**Needs** — [`controller_state_attack_fire.h`](controller_state_attack_fire.h.md) · [`controller_animation.h`](controller_animation.h.md) · [`controller_direction.h`](controller_direction.h.md) · [`controller.h`](controller.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_state_attack_fire.h`](controller_state_attack_fire.h.md)
**Tier floor** — T3: a timed predicate over remembered perception; no layout, no device, no budget of its own

## Purpose

One leaf state of the controller's brain. It exists to give the creature a *deliberate* shot:
the controller stops moving, turns its body and head onto the enemy, and drops its psychic
attack's own delay to zero so the attack lands during this window instead of whenever the
generic cooldown next permits. Everything about the state is the pair of predicates that open
and close that window; the body of the state does almost nothing.

Currently **unreachable in play** — nothing registers this state as a substate of any manager
(see [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md)). It is written,
compiled and dead. Treat it as a specification of an intended behaviour rather than as shipped
behaviour.

## State

```text
RECORD ControlFireState
  time_started           : int   # when this activation began
  time_state_last_execute: int   # when the last activation ended; 0 before the first
```

The second field is the only thing that survives a deactivation, and it is what enforces the
gap between shots. It is reset only by a full re-init of the brain, not by entering the state.

## `CStateControlFire`

**Contract** — a leaf state in the creature state contract (enter / execute each tick / leave,
plus two predicates the parent consults). Entering suppresses the owner's psychic-fire delay;
leaving restores it, by either exit path, and stamps the cooldown clock. Executing aims body and
head and requests the standing-torso, stealth-legs animation pairing. Allocates nothing; never
blocks.

**Invariants** — the psychic-fire delay is restored on *every* exit, ordinary or forced; a rebuild
that restores it only on the clean path leaves the creature permanently fast-firing after any
interruption. The cooldown stamp is written on both exits for the same reason.

```text
FUNCTION may_start() -> bool
  IF NOT sees_enemy_now()                   RETURN false
  IF distance_to(enemy) < MIN_ENEMY_DISTANCE RETURN false   # too close: this is a ranged act
  IF time_state_last_execute + EXECUTE_DELAY > now() RETURN false
  RETURN true

FUNCTION is_finished() -> bool
  # any of: lost sight, took a hit, enemy closed the gap, or the window expired
  RETURN NOT sees_enemy_now()
      OR was_hit_recently()
      OR distance_to(enemy) < MIN_ENEMY_DISTANCE
      OR time_started + STATE_MAX_TIME < now()

FUNCTION execute()
  face_body_toward(enemy)
  face_head_toward(head_position_of(enemy))   # aim at the head, not the origin
  set_body_pairing(torso = idle, legs = stealth)
```

**Notes** — the three numbers are authored *in code*, not in a configuration section, which makes
them the exception in this chapter: a minimum enemy distance of 10 world units, a maximum window
of 3 seconds, and 5 seconds between windows. The start and finish predicates share the distance
test, so the state cannot start and immediately end; the shared constant is what guarantees that,
and a rebuild that gives the two tests different thresholds introduces a flicker.

Looking at the enemy's *head* rather than its position is not cosmetic: the controller's psychic
attack is presented to the player as eye contact, and the head-look channel is a separate
direction consumer from the body facing.
