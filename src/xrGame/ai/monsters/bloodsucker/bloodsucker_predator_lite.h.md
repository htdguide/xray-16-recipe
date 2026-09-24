# src/xrGame/ai/monsters/bloodsucker/bloodsucker_predator_lite.h

> Declares the reactive stalking loop — the same three nodes as the full predator, but re-selected each cycle from whether the enemy can currently see the creature.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_predator_lite_inline.h`](bloodsucker_predator_lite_inline.h.md)
**Used by** — [`bloodsucker_attack_state_hide_inline.h`](bloodsucker_attack_state_hide_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](bloodsucker_predator_lite_inline.h.md)
**Tier floor** — T3: a declaration over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_predator_lite_inline.h`](bloodsucker_predator_lite_inline.h.md). Its only user is the mid-combat withdrawal composite, which is itself unregistered, so nothing in the shipped build reaches it. See [`bloodsucker_attack_state_hide_inline.h`](bloodsucker_attack_state_hide_inline.h.md).

Despite the name it is not a reduced [`bloodsucker_predator.h`](bloodsucker_predator.h.md): it holds different fields, selects differently, and completes differently. The two share only their node set and their cover-picking routine.

## `BloodsuckerPredatorLiteState`

A composite state holding the navigation vertex it has claimed from its pack and whether it currently has the creature frozen. It overrides entry, substate selection, both exits, the completion test, the parameter fill and the forced-restart hook.
