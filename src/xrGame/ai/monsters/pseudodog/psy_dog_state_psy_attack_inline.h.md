# src/xrGame/ai/monsters/pseudodog/psy_dog_state_psy_attack_inline.h

> The psi dog's "attack": a container with exactly one child, re-entered forever — the dog's contribution to the fight is to keep relocating, and the phantoms do the fighting.

**Needs** — [`psy_dog_state_psy_attack.h`](psy_dog_state_psy_attack.h.md) · [`psy_dog_state_psy_attack_hide.h`](psy_dog_state_psy_attack_hide.h.md) · [`../state_defs.h`](../state_defs.h.md)
**Used by** — [`psy_dog_state_psy_attack.h`](psy_dog_state_psy_attack.h.md)
**Tier floor** — T3: a one-child container

## Purpose

Five lines of substance, and they are worth reading because of what they imply. The psi dog's
attack state registers one child — the hide move — and its selector always chooses that child.
There is no second branch, no condition, no alternative.

## `CStatePsyDogPsyAttack` — construction

**Contract** — registers the hide move under the identifier the chapter uses for "hide in
cover", so that a debug dump of a psi dog in combat reads as the same family as any other
creature taking cover. Nothing else is registered.

## `reselect_state`

**Contract** — selects the hide move. Unconditionally, every time the selector is reached.

```text
FUNCTION reselect_state()
  select_state(hide_in_cover)
```

**Invariants** — combined with the container tick in [`../state_inline.h`](../state_inline.h.md),
this makes a loop with no exit: the hide move completes on arrival, the container clears its
choice, the next tick reselects the same child, and the child's entry picks a *new* cover
point. So the psi dog under attack runs from cover to cover indefinitely, and a creature that
"attacks" by relocating forever is exactly the design — the parent brain switches out of this
state only when the psi dog's phantom population recovers, which is decided in the dog itself,
not here.

**Notes** — because `select_state` is idempotent and the child is only ever re-entered *after*
completing, the destination changes once per arrival rather than once per tick. That is the
whole reason the destination is stored in the child rather than recomputed: the loop is driven
by the completion test, not by the selector.
