# src/xrGame/ai/monsters/states/state_look_unprotected_area.h

> Declares the leaf state that makes a cornered creature turn its back to cover and face the open ground it would have to flee across.

**Needs** — [`state.h`](../state.h.md) · [`state_data.h`](state_data.h.md) · [`state_look_unprotected_area_inline.h`](state_look_unprotected_area_inline.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../../../../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — [`group_state_panic_inline.h`](../group_states/group_state_panic_inline.h.md) · [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md) · [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md) · [`state_look_unprotected_area_inline.h`](state_look_unprotected_area_inline.h.md)
**Tier floor** — T3: a declaration, plus one query against the level's cover data

## Purpose

Declares the surface implemented in
[`state_look_unprotected_area_inline.h`](state_look_unprotected_area_inline.h.md).

## State

```text
RECORD LookUnprotectedAreaState
  data         : StateAction     # action, modifiers, sound, timeout
  target_point : vector3         # computed once at entry, then fixed
```

## Exported units

- **the face-the-open-ground state** — entry (computes the direction once), per-tick
  execution, and the same two-mode completion test as the turn-to-point state.

**Notes** — used by the panic behaviour and by the reaction to a dangerous sound. It is
the visible half of a creature deciding it is trapped.
