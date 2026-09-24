# src/xrGame/ai/monsters/states/monster_state_attack_melee_inline.h

> Biting: hold position, turn to face, and turn at two different speeds depending on whether you are already facing the right way.

**Needs** — [`monster_state_attack_melee.h`](monster_state_attack_melee.h.md)
**Used by** — [`monster_state_attack_melee.h`](monster_state_attack_melee.h.md)
**Tier floor** — T3: one action, one facing request

## Purpose

Six lines of behaviour with one genuine decision in them: the two-speed turn.

## `execute`

**Contract** — one tick. Sets the attack action, issues a facing request at the enemy, and sets
the aggressive state vocalisation. Never moves the creature; melee is stationary by
construction.

```text
FUNCTION execute()
  set_action(attack)

  IF I am already facing my enemy within 60 degrees
    face_target(enemy, over 800 milliseconds)         # a slow, readable tracking turn
  ELSE
    face_target(enemy, immediately, tolerance 15 degrees)   # snap round

  state_sound = aggressive
```

**Invariants** — the asymmetry is the whole point and it is not obvious from the call shape. A
creature that is *roughly* facing its target tracks it slowly over most of a second, which reads
as a predator following its prey's movements and, crucially, gives the player a window to move
out of the bite arc. A creature that is badly misaligned — the target has got behind it — turns
as fast as it can, with a fifteen-degree tolerance rather than none, so it does not overshoot.

A rebuild that uses one turn speed gets either a creature that can never be outmanoeuvred (fast)
or one that cannot recover when flanked (slow).

## `check_start_conditions`

**Contract** — melee may begin when the creature's melee checker admits it **and** the enemy is
visible right now. Both, not either.

**Invariants** — the visibility requirement is what stops a creature from biting a target it can
only sense through a wall — which is reachable, since some creatures perceive at range without
line of sight. Without it, such a creature would stand next to a partition attacking it.

## `check_completion`

**Contract** — melee ends when the melee checker says to stop. The checker's stop distance is
*larger* than its start distance, which is where the engagement hysteresis lives; see
[`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) for how the parent chain
relies on it.

**Notes** — visibility is *not* re-checked on completion, only on start. A creature that has
engaged keeps biting at a target that has broken line of sight, until distance alone separates
them. That asymmetry is deliberate — losing sight of something you are already biting should not
stop you — but it is the kind of thing a rebuild symmetrises by accident.
