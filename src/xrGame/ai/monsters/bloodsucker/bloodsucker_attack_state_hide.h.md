# src/xrGame/ai/monsters/bloodsucker/bloodsucker_attack_state_hide.h

> Declares the two-step mid-combat withdrawal: run to a reserved covered spot, then stalk from it.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_attack_state_hide_inline.h`](bloodsucker_attack_state_hide_inline.h.md)
**Used by** — [`bloodsucker_attack_state_hide_inline.h`](bloodsucker_attack_state_hide_inline.h.md)
**Tier floor** — T3: a declaration over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_attack_state_hide_inline.h`](bloodsucker_attack_state_hide_inline.h.md). Nothing in the shipped build constructs it — it was the withdrawal substate of the creature's own attack composite, which is itself never registered. See [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md).

## `BloodsuckerAttackHideState`

A composite state holding one field — the navigation vertex it has reserved from its pack — and overriding entry, substate selection, both exits, the completion test, the parameter fill and the forced-restart hook.
