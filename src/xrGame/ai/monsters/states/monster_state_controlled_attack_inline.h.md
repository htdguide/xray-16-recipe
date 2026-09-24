# src/xrGame/ai/monsters/states/monster_state_controlled_attack_inline.h

> Four one-line overrides that turn ordinary combat into commanded combat: force the enemy manager to report the controller's chosen target, then let the shared attack chain run untouched.

**Needs** — [`monster_state_controlled_attack.h`](monster_state_controlled_attack.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md)
**Used by** — [`monster_state_controlled_attack.h`](monster_state_controlled_attack.h.md)
**Tier floor** — T3: an override around an inherited behaviour

## Purpose

The cleanest example in the chapter of extending a behaviour by changing what it *perceives*
rather than what it *does*. The creature's enemy manager normally selects a target from memory
and danger weighting; forcing it makes it report one particular entity regardless. Everything
downstream — the approach, the melee hysteresis, the charge, the flight rules — then operates on
the commanded target with no knowledge that it was commanded.

## `initialize` and `execute`

**Contract** — both force the enemy manager onto the target the controller wrote, then defer to
the shared attack state. Entry forces it once; the tick forces it again, every tick.

**Invariants** — re-forcing every tick is necessary, not redundant: the enemy manager
re-evaluates on its own schedule and would otherwise drift back to its own choice between ticks
of this state. The cost is one assignment; the alternative is a puppet that intermittently
attacks whatever it likes.

**Notes** — forcing an enemy also suppresses the enemy manager's *danger verdict*, which several
creatures' outer selectors switch on with no default arm. That is the latent hazard recorded in
[`../state_defs.h`](../state_defs.h.md) and in the creature brains: a controlled creature is
precisely the case where the verdict is absent, and it is only the "am I controlled" branch
running *first* in those selectors that keeps the hazard unreachable.

## `finalize` and `critical_finalize`

**Contract** — both release the forced enemy after deferring to the base. **Both**, not one: a
creature released from control mid-fight must get its own target selection back, and a creature
whose combat merely completed must too.

**Invariants** — the release is the only teardown this state has, and forgetting it on the
interrupted path would leave the creature permanently fixated on whatever the controller last
named — including after the controller died.

## `get_enemy`

**Contract** — reads the target the controller wrote onto this creature's controlled-entity
facet and narrows it to a living thing. Returns nothing when the written object is not alive,
which is the case the container in
[`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md) catches before this
state is ever selected.
