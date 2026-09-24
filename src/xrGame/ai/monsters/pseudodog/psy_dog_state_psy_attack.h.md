# src/xrGame/ai/monsters/pseudodog/psy_dog_state_psy_attack.h

> Declares the psy dog's illusion attack: a one-node composite whose only job is to get the real animal out of sight.

**Needs** — [`state.h`](../state.h.md) · [`psy_dog_state_psy_attack_inline.h`](psy_dog_state_psy_attack_inline.h.md)
**Used by** — [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md) · [`psy_dog_state_psy_attack_inline.h`](psy_dog_state_psy_attack_inline.h.md)
**Tier floor** — T3: a declaration over the shared state contract

## Purpose

Declares the surface implemented in [`psy_dog_state_psy_attack_inline.h`](psy_dog_state_psy_attack_inline.h.md). It is a composite with a single child, which is an odd shape until you read it as a placeholder: the attack was expected to grow more phases and was given a composite from the start. Only the hiding phase was ever written, and the illusions themselves are spawned by the creature rather than by any state here.

Registered as the psy dog's psychic-attack global state; see [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md).

## `PsyDogPsyAttackState`

A composite state with no fields, overriding only substate selection — which unconditionally selects its single child — and reference cleanup.
