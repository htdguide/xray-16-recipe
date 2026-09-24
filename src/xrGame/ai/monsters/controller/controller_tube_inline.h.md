# src/xrGame/ai/monsters/controller/controller_tube_inline.h

> Yield the creature to its psychic-attack ability and stand still until the ability says it is
> finished.

**Needs** — [`controller_tube.h`](controller_tube.h.md) · [`controller_psy_hit.h`](controller_psy_hit.h.md) · [`controller.h`](controller.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_tube.h`](controller_tube.h.md)
**Tier floor** — T3: a delegation with two predicates

## Purpose

The controller's long-range psychic attack is not written as a state; it is an *ability* — a
separate component with its own animation, effect and lifetime, driven through the creature's
ability-control channel. This file is the adapter that lets the state machine own such an
ability: enter it, and the ability takes over; it finishes when the ability finishes.

The pattern is worth naming because it recurs: an ability that must run to completion is wrapped
in a state whose only job is to hold the brain still while the ability owns the body. A rebuild
that folds abilities into states loses the ability's independent lifetime and its ability to be
triggered from outside the brain.

**Currently unreachable in play**: no state manager registers it, and the composite that includes
its header never instantiates it.

## State

`Stateless.`

## `CStateControllerTube`

**Contract** — execute activates the creature's first custom ability channel and requests the
standing idle action, every tick, for as long as the state is active; it never inspects the
ability's progress. The start predicate is a conjunction of two independent refusals: the enemy
must have been continuously visible for at least one second, and the ability must itself say its
own preconditions hold. Finishing is entirely the ability's call.

**Invariants** — the state does not own the ability's lifetime; it observes it. A rebuild must not
stop the ability on exit, because the exit *is* the ability stopping.

```text
FUNCTION execute()
  activate_ability(custom_channel_1)
  request_action(stand_idle)

FUNCTION may_start() -> bool
  IF continuous_sight_duration(enemy) < SEE_ENEMY_DURATION  RETURN false
  RETURN psychic_attack.may_start()

FUNCTION is_finished() -> bool
  RETURN NOT psychic_attack.is_active()
```

**Notes** — the one-second continuous-sight requirement is the whole tactical content of the
state. It is a *continuous* duration, not "seen within the last second": a creature that catches
a glimpse of its enemy through a doorway cannot open with this attack, and one that has been
staring for a second can. That is what makes the attack readable to the player as a build-up
rather than an ambush.

Asking the ability for its own start conditions rather than duplicating them here is the right
split: the ability knows about its cooldown, its animation availability and its effect's
preconditions, none of which the brain should model.
