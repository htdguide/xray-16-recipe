# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_execute.h

> Declares the leaf state that performs the feed itself: seize the player, hold, drain, release.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker_vampire_execute_inline.h.md)
**Used by** — [`bloodsucker_attack_state_inline.h`](bloodsucker_attack_state_inline.h.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker_vampire_execute_inline.h.md) · [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md)
**Tier floor** — T2: behaviour over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_vampire_execute_inline.h`](bloodsucker_vampire_execute_inline.h.md). It is the deepest node of the vampire behaviour tree and the only one that takes the player's controls away, so it is split out from the tree that selects it.

## `VampireExecuteState`

A leaf state over the shared state contract. Its own state is a four-step phase marker (prepare, continue, fire, wait for the animation to finish, done), the time the hold began, and whether the screen effects have been started yet. It fills in `initialize`, `execute`, `finalize`, `critical_finalize`, `check_start_conditions` and `check_completion`; everything else it inherits.
