# src/xrGame/ai/monsters/states/state_hit_object_inline.h

> Implements the unused object-shove: pick one physics object inside a cone in front of the creature and, half a second in, push it away.

**Needs** — [`state_hit_object.h`](state_hit_object.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`state_hit_object.h`](state_hit_object.h.md)
**Tier floor** — T3: a proximity query, two angle tests and one impulse

## Purpose

The behaviour it would produce, if any creature selected it: a creature stops, plays its
corpse-inspection animation, and after a short wind-up knocks a loose object aside with an
impulse scaled to the object's mass — so a barrel and a crate fly the same way regardless
of weight.

**Nothing selects this state.** See [`state_hit_object.h`](state_hit_object.h.md).

## `CStateMonsterHitObject`

**Contract** — the start condition scans for physics-bearing objects within the creature's
own radius less half a metre, and accepts the first one whose direction lies inside a
30-degree cone around the creature's facing in *both* yaw and pitch. It records that object
as the target. Execution holds the creature in its idle stance with the corpse-inspection
animation modifier set, and applies exactly one impulse once half a second has elapsed
since entry. The state expires one second after entry regardless.

```text
FUNCTION check_start_conditions() -> bool
  target = none
  nearby = object_space.objects_within(object.position, object.radius - 0.5, exclude = object)
  FOR EACH candidate IN nearby
    IF candidate has no physics body THEN CONTINUE
    to_candidate = candidate.position - object.position
    (my_yaw, my_pitch)   = heading_and_pitch(object.direction)
    (its_yaw, its_pitch) = heading_and_pitch(to_candidate)
    IF its_yaw   NOT WITHIN my_yaw   +/- 30 degrees THEN CONTINUE
    IF its_pitch NOT WITHIN my_pitch +/- 30 degrees THEN CONTINUE
    target = candidate
    RETURN true
  RETURN false

FUNCTION execute()
  object.set_action(stand_idle)
  object.animation.set_modifiers(check_corpse)
  IF NOT hitted AND now() > time_state_started + 500 milliseconds
    hitted = true
    # the push is away from the creature, biased by where the creature is looking,
    # so a shove delivered mid-turn throws the object sideways rather than straight back
    direction = normalize((target.position - object.position) + object.direction)
    target.physics.apply_impulse(direction, 20 * target.physics.mass)

FUNCTION check_completion() -> bool
  RETURN now() > time_state_started + 1000 milliseconds
```

**Invariants** — the target is selected in the start condition and used in execution, so a
rebuild must guarantee the start condition runs before every activation, and must handle
the target being destroyed inside the one-second window. The original does neither
explicitly; it survives because the state is never entered.

**Notes** — the acceptance cone rejects on yaw *and* pitch against the same half-angle,
which means a creature cannot shove something at its feet even when it is directly ahead.
Whether that was intended is not recoverable.
