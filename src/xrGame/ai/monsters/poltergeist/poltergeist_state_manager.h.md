# src/xrGame/ai/monsters/poltergeist/poltergeist_state_manager.h

> Declares the poltergeist's brain: seven registered states and a selector with only two live outcomes.

**Needs** — [`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md)
**Used by** — [`poltergeist.cpp`](poltergeist.cpp.md) · [`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md)
**Tier floor** — T3: a selector over a registered state set

## Purpose

Declares the surface implemented in
[`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md). It is the root of the
poltergeist's state tree, in the shape every creature's root takes — see
[`../monster_state_manager.h`](../monster_state_manager.h.md).

## Exported units

- construction — registers the seven states.
- `reinit` — resets the three attack cooldown clocks.
- `execute` — the selector.
- `forget_entity` — forwards to the base.

It also holds three cooldown timestamps, one per kind of attack, which belong to a scare
routine that is not live. See the implementation.
