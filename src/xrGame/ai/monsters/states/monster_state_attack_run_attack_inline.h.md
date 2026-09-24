# src/xrGame/ai/monsters/states/monster_state_attack_run_attack_inline.h

> The charge-through: the creature keeps running and sets a marker on its run animation that makes the clip carry a hit. Whether it connects is the animation's business, not the state's.

**Needs** — [`monster_state_attack_run_attack.h`](monster_state_attack_run_attack.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md)
**Used by** — [`monster_state_attack_run_attack.h`](monster_state_attack_run_attack.h.md)
**Tier floor** — T3: an animation parameter and two gate tests

## Purpose

The interesting thing here is the inversion of responsibility. The state does not decide that a
hit happened; it sets a *special parameter* on the running animation, and the animation layer
fires the hit at the frame the clip says so and stamps the creature. The state only watches for
that stamp. Damage timing is therefore authored in the animation, not in the AI.

## `initialize`

**Contract** — clears the creature's last-successful-hit stamp, which is the state's only
completion signal.

## `execute`

**Contract** — one tick. Sets the running action, the aggressive vocalisation, and the
attack-while-running marker on the animation. Nothing else — no path request, so the charge
continues along whatever route the approach was already following.

**Invariants** — inheriting the previous route rather than issuing a new one is what makes the
charge a continuation rather than a new manoeuvre, and it is why the state can only be entered
from the approach (see
[`monster_state_attack_inline.h`](monster_state_attack_inline.h.md)).

## `check_start_conditions`

**Contract** — whether the charge may begin.

```text
FUNCTION check_start_conditions() -> bool
  d = distance_to_enemy_per_the_melee_checker()
  IF d > creature.run_attack_start_distance     RETURN false   # authored per creature
  IF d < melee_checker.minimum_distance         RETURN false   # too close to build up
  IF NOT facing the enemy within 30 degrees     RETURN false
  RETURN true
```

**Invariants** — the gate is a *window*, not a threshold: too far and the creature has not
closed yet, too near and there is no room to charge through. The thirty-degree cone is what
makes the charge read as deliberate — a creature does not charge sideways.

**Notes** — the routine also computes a point one authored charge-distance straight ahead of the
creature and then does nothing with it. The path-build test that would have used it — "can I
actually run that far?" — is **commented out**, together with the line that would have enabled
the resulting route. So **the charge starts without any check that there is room for it**, and a
creature can charge into a wall. That is a live defect with a visible symptom, and a rebuild
should restore the test rather than reproduce the omission.

## `check_completion`

**Contract** — the charge is over when the creature is no longer moving along its route, or when
the creature's last-successful-hit stamp has been set by the animation layer.

**Invariants** — the two conditions are "I missed and ran out of run" and "I connected". Either
ends it, and in both cases the parent chain falls back to the approach-or-melee pair on the next
tick.
