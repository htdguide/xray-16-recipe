# src/xrGame/ai/monsters/controller/controller_state_attack_fire.h

> Declares the controller's stand-and-stare psychic attack. Included by the attack composite, but never instantiated.

**Needs** — [`state.h`](../state.h.md) · [`controller_state_attack_fire_inline.h`](controller_state_attack_fire_inline.h.md)
**Used by** — [`controller_state_attack_fire_inline.h`](controller_state_attack_fire_inline.h.md) · [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`controller_state_attack_fire_inline.h`](controller_state_attack_fire_inline.h.md), which carries the contracts.

The controller's attack composite includes this header and registers three substates — approach the home point, run at the enemy, and strike — but not this one. Nothing else constructs it either. The creature's psychic attack still fires in the shipped build; it is driven from the creature's own ability rather than from a behaviour state, which is why removing the state from the composite did not remove the attack from the game.

The state identifier it was written for is not orphaned, though: the *group* attack composite in [`group_states/`](../group_states/README.md) registers a different, generic state under it. A rebuilder searching by state identifier will find that one and must not mistake it for this.

## `ControllerFireState`

A leaf state holding two timestamps — when it began and when it was last updated — and overriding reinitialisation, entry, update, both exits, and both the start and completion tests.
