# src/xrGame/ai/monsters/states/state_move_to_restrictor.h

> Declares the corrective state that runs when a creature finds itself outside the volume it is allowed to be in.

**Needs** — [`state.h`](../state.h.md) · [`state_move_to_restrictor_inline.h`](state_move_to_restrictor_inline.h.md)
**Used by** — [`burer_state_attack_inline.h`](../burer/burer_state_attack_inline.h.md) · [`group_state_rest_inline.h`](../group_states/group_state_rest_inline.h.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`state_move_to_restrictor_inline.h`](state_move_to_restrictor_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`state_move_to_restrictor_inline.h`](state_move_to_restrictor_inline.h.md). It holds no
parameter record — everything it needs comes from the creature's own restriction set.

Selected by the resting behaviour of several creatures and by the burer's attack, always
under the same identifier, always as the first thing checked. It is the creature layer's
answer to "a script moved the restrictor, or teleported me, and now I am somewhere I may
not be".

## Exported units

- **the return-to-permitted-space state** — a start condition that is exactly "I am
  outside", an entry that picks the destination, a fixed execution, and a completion test
  that is exactly "I am inside again".
