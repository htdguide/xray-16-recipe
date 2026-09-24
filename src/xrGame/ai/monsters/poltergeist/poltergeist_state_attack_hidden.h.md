# src/xrGame/ai/monsters/poltergeist/poltergeist_state_attack_hidden.h

> Declares the poltergeist's only attack state: circle the target at a distance while the abilities do the damage.

**Needs** — [`poltergeist_state_attack_hidden_inline.h`](poltergeist_state_attack_hidden_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`poltergeist_state_attack_hidden_inline.h`](poltergeist_state_attack_hidden_inline.h.md) · [`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md)
**Tier floor** — T3: a route target chosen by sampling a circle

## Purpose

Declares the state implemented in
[`poltergeist_state_attack_hidden_inline.h`](poltergeist_state_attack_hidden_inline.h.md). It
has one registered child — the state that sends the creature back inside its home — and
otherwise no sub-states: the "attack" is entirely a movement pattern, because the damage comes
from the ability running on its own clock.

## Exported units

- `enter` — prepare the route builder and reset the orbit.
- `run` — the selector-and-movement step.
- `home_claim` — whether the return-home child should take over.
- `forget_entity` — forwards to the base.

It carries the orbit state: which way round the creature is going, when to reconsider that, the
current radius as a fraction of the authored one, and the target point and vertex.
