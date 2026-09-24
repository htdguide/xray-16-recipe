# src/xrGame/ai/monsters/controller/controller_tube.h

> Declares the wrapper state that hands the creature over to the psychic-attack ability,
> implemented in [`controller_tube_inline.h`](controller_tube_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`controller_tube_inline.h`](controller_tube_inline.h.md)
**Used by** — [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md) · [`controller_tube_inline.h`](controller_tube_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the three-method surface of the thinnest state in the chapter: it does not decide anything,
it activates an ability and reports when that ability is done. Split from its body only because
C++ splits templates that way.

## `CStateControllerTube`

- **execute** — activate the custom ability channel and stand still
- **may_start** — the ability itself must agree, and the enemy must have been in sight long enough
- **is_finished** — the ability is no longer running

Contracts are in the implementation twin.
